---
layout:     post
title:      gRPC 应用篇之自带组件
subtitle:
date:       2020-02-04
author:     pandaychen
header-img:
catalog: true
category:   false
tags:
    - gRPC
---

##	0x01	Metadata 应用
&emsp;&emsp;gRPC 的 [Metadata 机制](https://github.com/grpc/grpc-go/blob/master/Documentation/grpc-metadata.md) 提供了一种类似 HTTP-Header 的机制（理解为 RPC 方法的 Header）。它的应用场景是：对于每一次的 RPC 调用中，都可能会有一些有用的数据，而这些数据就可以通过 metadata 来传递。metadata 是以 key-value 的形式存储数据的，其中 key 是 `string` 类型，而 value 是 `[]string` 类型。metadata 使得 client 和 server 能够为对方提供关于本次调用的一些信息，就像一次 http 请求的 RequestHeader 和 ResponseHeader 一样，HTTP 中 header 的生命周周期是一次 HTTP 请求，那么 metadata 的生命周期就是一次 RPC 调用。

#### Metadata 的应用场景
1.	采集 CPU 数据, 由服务端返回给客户端
2.	对 RPC 方法做认证：例如，每次访问（请求）需要认证的数据，如 JWT 的 token、CSRF 的 token、cookies 等


#### Metadata 实战
例如，实现每次 RPC 调用后，都采集服务端的系统（如 CPU、内存）数据，如下代码所示：
```golang
func (s *Server) stats() grpc.UnaryServerInterceptor {
	return func(ctx context.Context, req interface{}, args *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (resp interface{}, err error) {
		resp, err = handler(ctx, req)
		// 采集系统数据
		if cpustat.Usage != 0 {
			// 每次客户端 RPC 请求，服务端都会计算 cpu 使用率，gRPC 客户端在 Pick 的 DoneInfo 中获取此值
			trailer := metadata.Pairs([]string{nmd.CPUUsage, strconv.FormatInt(int64(cpustat.Usage), 10)}...)
			// 每次 rpc 请求时，放在 tailer，上报至 discovery
			grpc.SetTrailer(ctx, trailer)
		}
		return
	}
}
```

##  0x02    拦截器链（Interceptor Chain）的封装
以 gRPC 的服务端拦截器实现为例，封装一个实用的拦截器链。该链满足如下特性：定义一个链，`ChainUnaryServer(one, two, three)` 按照 `one`、`two`、`three` 的顺序执行。

#### 封装 Server 的结构
```golang
//grpc-server 核心结构
type Server struct {
    ...
	server   *grpc.Server	//
	handlers []grpc.UnaryServerInterceptor	// 封装拦截器数组
}
```

#### 修改 grpc.UnaryInterceptor 传入的参数 UnaryServerInterceptor，完成链式功能
先看下 `grpc.UnaryInterceptor` 这个方法的 [定义](https://godoc.org/google.golang.org/grpc#UnaryInterceptor)，该方法的唯一参数是 `grpc.UnaryServerInterceptor`
```GOLANG
func UnaryInterceptor(i UnaryServerInterceptor) ServerOption
```
而 `grpc.UnaryServerInterceptor` 的 [定义](https://godoc.org/google.golang.org/grpc#UnaryServerInterceptor) 是一个方法原型，这是我们需要实现的（链式 interceptor）
```GOLANG
type UnaryServerInterceptor
func(ctx context.Context, req interface{}, info *UnaryServerInfo, handler UnaryHandler) (resp interface{}, err error)
```
实现链式拦截器的功能：
```GOLANG
// 拦截器链实现
func (s *Server) interceptor(ctx context.Context, req interface{}, args *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (interface{}, error) {
	var (
		i     int
		chain grpc.UnaryHandler
	)

	n := len(s.handlers)
	if n == 0 {
		return handler(ctx, req)
	}

	chain = func(ic context.Context, ir interface{}) (interface{}, error) {
		if i == n-1 {
			return handler(ic, ir)
		}
		i++
		return s.handlers[i](ic, ir, args, chain)
	}

	return s.handlers[0](ctx, req, args, chain)
}
```

####   完成 grpc.UnaryInterceptor 初始化
接下来，将上一步实现的 `UnaryServerInterceptor`，当做 `grpc.UnaryInterceptor()` 方法的参数传入，这样，就使用我们自定义的链式拦截器了。
```GOLANG
// NewServer returns a new blank Server instance with a default server interceptor.
// 初始化 Server
func NewServer(conf *ServerConfig, opt ...grpc.ServerOption) (s *Server) {
	if conf == nil {
		if !flag.Parsed() {
			fmt.Fprint(os.Stderr, "[warden] please call flag.Parse() before Init warden server, some configure may not effect\n")
		}
		conf = parseDSN(_grpcDSN)
	} else {
		fmt.Fprintf(os.Stderr, "[warden] config is Deprecated, argument will be ignored. please use -grpc flag or GRPC env to configure warden server.\n")
	}
	s = new(Server)
	if err := s.SetConfig(conf); err != nil {
		panic(errors.Errorf("warden: set config failed!err: %s", err.Error()))
	}
	keepParam := grpc.KeepaliveParams(keepalive.ServerParameters{
		MaxConnectionIdle:     time.Duration(s.conf.IdleTimeout),
		MaxConnectionAgeGrace: time.Duration(s.conf.ForceCloseWait),
		Time:                  time.Duration(s.conf.KeepAliveInterval),
		Timeout:               time.Duration(s.conf.KeepAliveTimeout),
		MaxConnectionAge:      time.Duration(s.conf.MaxLifeTime),
	})
	opt = append(opt, keepParam, grpc.UnaryInterceptor(s.interceptor))  //
	s.server = grpc.NewServer(opt...)
	s.Use(s.recovery(), s.handle(), serverLogging(conf.LogFlag), s.stats(), s.validate())
	s.Use(ratelimiter.New(nil).Limit())
	return
}
```

####	定义向链式拦截器数组添加拦截器实现的方法 Use
```GOLANG
// Use attachs a global inteceptor to the server.
// For example, this is the right place for a rate limiter or error management inteceptor.

// 设置一个全局的拦截器，其实就是修改 s 自带的拦截器数组
func (s *Server) Use(handlers ...grpc.UnaryServerInterceptor) *Server {
	finalSize := len(s.handlers) + len(handlers)
	if finalSize >= int(_abortIndex) {
		panic("warden: server use too many handlers")
	}
	mergedHandlers := make([]grpc.UnaryServerInterceptor, finalSize)
	copy(mergedHandlers, s.handlers)
	copy(mergedHandlers[len(s.handlers):], handlers)
	s.handlers = mergedHandlers		// 更新自带的 handlers，数组
	return s
}
```

####	最后一步，使用 Use 方法

```GOLANG
// NewServer returns a new blank Server instance with a default server interceptor.
// 初始化 Server
func NewServer(conf *ServerConfig, opt ...grpc.ServerOption) (s *Server) {
	...
	s = new(Server)
	if err := s.SetConfig(conf); err != nil {
		panic(errors.Errorf("warden: set config failed!err: %s", err.Error()))
	}
	keepParam := grpc.KeepaliveParams(keepalive.ServerParameters{
		MaxConnectionIdle:     time.Duration(s.conf.IdleTimeout),
		MaxConnectionAgeGrace: time.Duration(s.conf.ForceCloseWait),
		Time:                  time.Duration(s.conf.KeepAliveInterval),
		Timeout:               time.Duration(s.conf.KeepAliveTimeout),
		MaxConnectionAge:      time.Duration(s.conf.MaxLifeTime),
	})
	opt = append(opt, keepParam, grpc.UnaryInterceptor(s.interceptor))  //
	s.server = grpc.NewServer(opt...)
	s.Use(s.recovery(), s.handle(), serverLogging(conf.LogFlag), s.stats(), s.validate())
	s.Use(ratelimiter.New(nil).Limit())
	...
	return
}
```

##	0x03	实现 gRPC 的一元拦截器
这里举几个项目中封装的拦截器例子：

####	计算 RPC 请求的耗时
```go
func logReqTime() grpc.UnaryClientInterceptor {
    return func(ctx context.Context, method string, req, reply interface{}, cc *grpc.ClientConn, invoker grpc.UnaryInvoker, opts ...grpc.CallOption) error {
		start := time.Now() // 取得调用前的时间

		// 取对端的地址
		// 使用 grpc.Peer() 取得调用 rpc 服务的地址
        p := peer.Peer{}
        if opts == nil {
            opts = []grpc.CallOption{grpc.Peer(&p)}
        } else {
            opts = append(opts, grpc.Peer(&p))
        }

        err := invoker(ctx, method, req, reply, cc, opts...)    // 调用 rpc 函数
        cost := time.Since(start)   // 取得调用函数的耗时
        fmt.Printf("method[%s] cost[%v]n", method, cost) // 打印耗时
        return err
    }
}
```
客户端调用，在 grpc.Dial() 方法中添加 Interceptor 中间件 logReqTime
```go
conn, err := grpc.Dial(address, grpc.WithInsecure(), grpc.WithUnaryInterceptor(logReqTime()))
```

执行结果如下：
```bash
method[/helloworld.Greeter/SayHello] call[[::1]:50051] cost[3.072303ms]
```


##	0x04	参考
-	[gRPC Metadata 文档](https://github.com/grpc/grpc-go/blob/master/Documentation/grpc-metadata.md)
-	[gRPC 的那些事 - interceptor](https://colobu.com/2017/04/17/dive-into-gRPC-interceptor/)

转载请注明出处，本文采用 [CC4.0](http://creativecommons.org/licenses/by-nc-nd/4.0/) 协议授权