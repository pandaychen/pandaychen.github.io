---
layout:     post
title:      Kratos 源码分析：超时（Timeout）传递
subtitle:   Context 的用法：Warden/Database 中的超时传递实现分析
date:       2020-06-20
author:     pandaychen
catalog:    true
tags:
    - Kratos
---

##  0x00    前言
本篇文章，了解下 Golang、gRPC 框架中的超时传递，在微服务中，超时和熔断、重试、Backoff 策略都是有关联的。在实际项目中，每一个 RPC 调用都应该有超时退出的能力，这是比较合理的 API 设计。在 [Context Deadlines and How to Set Them](https://engineering.grab.com/context-deadlines-and-how-to-set-them) 中总结了超时的要点：

1.  Return an error. This is the simplest, but unless you know there is error handling upstream, this can actually deliver the worst user experience.
2.  Return a fallback value. We can return a default value, a cached value, or fall back to a simpler computed value. Depending on the circumstances, this can offer a better user experience.
3.  Retry. In the best case, a retry will succeed and deliver the intended response to the caller, albeit with the added timeout delay. However, there are other complexities to consider for retries to be effective. For a full discussion on this topic, see Circuit Breaker vs Retries Part 1and Circuit Breaker vs Retries Part 2.

##  0x01    回顾 Context
`Context` 接口如下：
```golang
// A Context carries a deadline, cancelation signal, and request-scoped values
// across API boundaries. Its methods are safe for simultaneous use by multiple
// goroutines.
type Context interface {
    // Done returns a channel that is closed when this Context is canceled
    // or times out.
    Done() <-chan struct{}

    // Err indicates why this context was canceled, after the Done channel
    // is closed.
    Err() error

    // Deadline returns the time when this Context will be canceled, if any.
    Deadline() (deadline time.Time, ok bool)

    // Value returns the value associated with key or nil if none.
    Value(key interface{}) interface{}
}
```
包含了四个方法：
-	`Done()`，返回一个 channel，当 times out 或者调用 cancel 方法时，将会 close 掉
-	`Err()`，返回一个错误，该 `Context` 为什么被取消掉
-	`Deadline()`，返回截止时间和设置标记 `ok`
-	`Value()`，返回值

`Context` 的四个派生方法：
```golang
func WithCancel(parent Context) (ctx Context, cancel CancelFunc)
func WithDeadline(parent Context, deadline time.Time) (Context, CancelFunc)
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc)
func WithValue(parent Context, key, val interface{}) Context
```

其中，有一个超时的生成方法 `func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc)` 正式我们本文需要用到的。

##  0x01    进程内传递

##  0x02    跨进程传递
在这篇博客 [gRPC 系列——grpc 超时传递原理](https://xiaomi-info.github.io/2019/12/30/grpc-deadline/)，介绍了跨进程（语言）的超时传递的场景。先给出结论：gRPC 框架确实是通过 HTTP2 HEADERS Frame 中的 grpc-timeout 字段来实现跨进程传递超时时间。


##  0x03	Warden 框架的超时传递实现
通过 `Shrink(c context.Context)` 方法来完成进程内的超时判定及传递：
```golang
// Shrink will decrease the duration by comparing with context's timeout duration
// and return new timeout\context\CancelFunc.
// 非常棒的实现！RPC 超时传递
func (d Duration) Shrink(c context.Context) (Duration, context.Context, context.CancelFunc) {
	//Deadline 方法是获取设置的截止时间的意思，第一个返回是截止时间，到了这个时间点，Context 会自动发起取消请求；第二个返回值 ok==false 时表示没有设置截止时间，如果需要取消的话，需要调用取消函数进行取消。
	if deadline, ok := c.Deadline(); ok {
		// 该 ctx 设置了截止时间
		if ctimeout := xtime.Until(deadline); ctimeout < xtime.Duration(d) {
			// deliver small timeout
			return Duration(ctimeout), c, func() {}
		}
	}
	// 未设置截止时间时，设置为当前时间
	// 更新 ctx 的超时时间并 fork（子孙）
	ctx, cancel := context.WithTimeout(c, xtime.Duration(d))
	return d, ctx, cancel
}
```

##	0x04	Warden 超时传递的应用

####	客户端超时

在 Warden 的 [`client.go`](https://github.com/go-kratos/kratos/blob/master/pkg/net/rpc/warden/client.go#L101) 中，关于超时设置的代码如下：
```golang
func (c *Client) handle() grpc.UnaryClientInterceptor {
	return func(ctx context.Context, method string, req, reply interface{}, cc *grpc.ClientConn, invoker grpc.UnaryInvoker, opts ...grpc.CallOption) (err error) {
		......
		// 熔断统计结果必须要在 rpc 调用最后
		defer onBreaker(brk, &err)
		var timeOpt *TimeoutCallOption
		for _, opt := range opts {
			var tok bool
			timeOpt, tok = opt.(*TimeoutCallOption)
			if tok {
				break
			}
		}
		if timeOpt != nil && timeOpt.Timeout > 0 {
			// 如果定义了超时配置，那么久使用定义的逻辑
			ctx, cancel = context.WithTimeout(nmd.WithContext(ctx), timeOpt.Timeout)
		} else {
			// 否则，使用 shrink 继承 CONTEXT 中的超时配置
			_, ctx, cancel = conf.Timeout.Shrink(ctx)
		}

		defer cancel()
		nmd.Range(ctx,
			func(key string, value interface{}) {
				if valstr, ok := value.(string); ok {
					gmd[key] = []string{valstr}
				}
			},
			nmd.IsOutgoingKey)
		......
}
```

####	服务端超时
服务端关于超时处理的代码在拦截器 [`server.handle()` 中](https://github.com/go-kratos/kratos/blob/master/pkg/net/rpc/warden/server.go#L85)：
```golang
func (s *Server) handle() grpc.UnaryServerInterceptor {
	return func(ctx context.Context, req interface{}, args *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (resp interface{}, err error) {
		var (
			cancel func()
			addr   string
		)
		s.mutex.RLock()
		conf := s.conf
		s.mutex.RUnlock()
		// get derived timeout from grpc context,
		// compare with the warden configured,
		// and use the minimum one
		timeout := time.Duration(conf.Timeout)
		if dl, ok := ctx.Deadline(); ok {
			ctimeout := time.Until(dl)
			if ctimeout-time.Millisecond*20 > 0 {
				ctimeout = ctimeout - time.Millisecond*20
			}
			if timeout > ctimeout {
				timeout = ctimeout
			}
		}
		ctx, cancel = context.WithTimeout(ctx, timeout)
		defer cancel()

		// get grpc metadata(trace & remote_ip & color)
		var t trace.Trace
		cmd := nmd.MD{}
		if gmd, ok := metadata.FromIncomingContext(ctx); ok {
			t, _ = trace.Extract(trace.GRPCFormat, gmd)
			for key, vals := range gmd {
				if nmd.IsIncomingKey(key) {
					cmd[key] = vals[0]
				}
			}
		}
		if t == nil {
			t = trace.New(args.FullMethod)
		} else {
			t.SetTitle(args.FullMethod)
		}

		if pr, ok := peer.FromContext(ctx); ok {
			addr = pr.Addr.String()
			t.SetTag(trace.String(trace.TagAddress, addr))
		}
		defer t.Finish(&err)

		// use common meta data context instead of grpc context
		ctx = nmd.NewContext(ctx, cmd)
		ctx = trace.NewContext(ctx, t)

		resp, err = handler(ctx, req)
		return resp, status.FromError(err).Err()
	}
}
```

##	0x05	思考
这里有个疑问，为何仅仅在 gRPC 的 client 端才嵌入超时传递的 `Shrink` 逻辑呢？而 Warden 的服务端却不需要 `Shrink` 逻辑？


##	0x06	Database 中超时传递的实现


##  0x07	参考
-   [gRPC 系列——grpc 超时传递原理](https://xiaomi-info.github.io/2019/12/30/grpc-deadline/)
-	[Golang 如何正确使用 Context](https://juejin.im/post/5d6b5dc3e51d4561ce5a1c94)
