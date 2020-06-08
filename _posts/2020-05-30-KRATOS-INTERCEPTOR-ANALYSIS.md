---
layout:     post
title:      Kratos 源码分析（2）：gRPC-Warden 拦截器（链）及实现
subtitle:   Kratos 框架分析
date:       2020-05-30
author:     pandaychen
header-img:
catalog: true
category:   false
tags:
    - Kratos
---

##  0x00 前言
&emsp;&emsp; 在 Warden 框架中，大量使用了拦截器来完成对 RPC 功能的增加及统一接口封装，本文从源码的角度来分析下 Warden 中的拦截器 Interceptor。


##	0x01	Warden 拦截器总览
1.	[logging.go](https://github.com/go-kratos/kratos/blob/master/pkg/net/rpc/warden/logging.go)：包含了 `UnaryServerInterceptor` 及 `UnaryClientInterceptor` 的实现，通用日志逻辑，其中加入了对 RPC 请求耗时的 metrics 上报（prometheus）
2.	[ratelimiter.go](https://github.com/go-kratos/kratos/blob/master/pkg/net/rpc/warden/ratelimiter/ratelimiter.go)：`UnaryServerInterceptor` 实现，在服务端实现限速逻辑
3.	[validate.go](https://github.com/go-kratos/kratos/blob/master/pkg/net/rpc/warden/validate.go)：`UnaryServerInterceptor` 实现，在服务端校验请求参数合法性
4.	[stats.go](https://github.com/go-kratos/kratos/blob/master/pkg/net/rpc/warden/stats.go)：`UnaryServerInterceptor` 实现，通过 `grpc.SetTrailer` 返回服务端的实时 CPU 使用率
5.	[metrics.go](https://github.com/go-kratos/kratos/blob/master/pkg/net/rpc/warden/metrics.go)：定义 `logging.go` 中的 prometheus 的 `vectors`
6.	[recovery.go](https://github.com/go-kratos/kratos/blob/master/pkg/net/rpc/warden/recovery.go)：包含了 `UnaryServerInterceptor` 及 `UnaryClientInterceptor`，用于 coredump 时捕获堆栈错误，打印堆栈信息

此外，在 [server.go](https://github.com/go-kratos/kratos/blob/master/pkg/net/rpc/warden/server.go) 及 [client.go](https://github.com/go-kratos/kratos/blob/master/pkg/net/rpc/warden/client.go) 中也包含了若干拦截器，这个放在后面单独的分析篇中再展开。


##	0x02	Warden-Server 端 Interceptor
gRPC 暴露了服务端、客户端两个拦截器接口，基于两个拦截器可以针对性的 ** 定制公共模块 ** 的封装代码。
* `grpc.UnaryServerInterceptor` 服务端拦截器
* `grpc.UnaryClientInterceptor` 客户端拦截器

####	gRPC 服务端拦截器
首先，看一下 `grpc.UnaryServerInterceptor` 的 [声明](https://github.com/grpc/grpc-go/blob/master/interceptor.go)，如下：

1.	`UnaryServerInfo` 结构：用于 `Server` 和 `FullMethod` 字段传递，`Server` 为 `gRPC server` 的对象实例，`FullMethod` 为 RPC 方法的全名
2.	`UnaryHandler` 方法：用于传递 `Handler`，就是基于 `proto` 文件 `service` 内声明而生成的方法
3.	`UnaryServerInterceptor` 方法：用于拦截 `handler` 方法，可在 `handler` 执行前后插入拦截代码（参数中的 `handler UnaryHandler`）

```golang
// UnaryServerInfo consists of various information about a unary RPC on
// server side. All per-rpc information may be mutated by the interceptor.
type UnaryServerInfo struct {
	// Server is the service implementation the user provides. This is read-only.
	Server interface{}
	// FullMethod is the full RPC method string, i.e., /package.service/method.
	FullMethod string
}

// UnaryHandler defines the handler invoked by UnaryServerInterceptor to complete the normal
// execution of a unary RPC. If a UnaryHandler returns an error, it should be produced by the
// status package, or else gRPC will use codes.Unknown as the status code and err.Error() as
// the status message of the RPC.
type UnaryHandler func(ctx context.Context, req interface{}) (interface{}, error)

// UnaryServerInterceptor provides a hook to intercept the execution of a unary RPC on the server. info
// contains all the information of this RPC the interceptor can operate on. And handler is the wrapper
// of the service method implementation. It is the responsibility of the interceptor to invoke handler
// to complete the RPC.
type UnaryServerInterceptor func(ctx context.Context, req interface{}, info *UnaryServerInfo, handler UnaryHandler) (resp interface{}, err error)
```

##	0x03	Warden-Client 端 Interceptor

####	gRPC 客户端拦截器
客户端拦截器 `grpc.UnaryClientInterceptor` 的声明 [在此](https://github.com/grpc/grpc-go/blob/master/interceptor.go)，和 `UnaryServerInterceptor` 类似，只不过，服务端的 RPC 方法叫 `handler`，客户端作为 RPC 调用方叫 `invoker`：

1.	`UnaryInvoker` 方法：表示客户端具体要发出的执行方法
2.	`UnaryClientInterceptor` 方法：用于拦截 `invoker` 方法，可在 `invoker` 执行前后插入拦截代码

```golang
// UnaryInvoker is called by UnaryClientInterceptor to complete RPCs.
type UnaryInvoker func(ctx context.Context, method string, req, reply interface{}, cc *ClientConn, opts ...CallOption) error

// UnaryClientInterceptor intercepts the execution of a unary RPC on the client. invoker is the handler to complete the RPC
// and it is the responsibility of the interceptor to call it.
// This is an EXPERIMENTAL API.
type UnaryClientInterceptor func(ctx context.Context, method string, req, reply interface{}, cc *ClientConn, invoker UnaryInvoker, opts ...CallOption) error
```

##	0x04	Warden 拦截器链（Chain）
为了减轻开发者对拦截器的依赖，gRPC 特意要求，无论服务端 or 客户端只能注册一个拦截器（官方的说法：Only one unary interceptor can be installed. The construction of multiple interceptors (e.g., chaining) can be implemented at the caller.），但是实际中，一个 Interceptor 肯定是不够的，所以需要对单拦截器进行扩展，那就是拦截器链。Warden 包针对服务端和客户端都封装了 Chain 的实现。

####	服务端 Chain
对于服务端，Warden 封装的代码 [在此](https://github.com/go-kratos/kratos/blob/master/pkg/net/rpc/warden/server.go#L263)

在服务端注册时，调用 `opt = append(opt, keepParam, grpc.UnaryInterceptor(s.interceptor))`，告诉 gRPC，拦截器 Chain 该如何调用，调用的代码如下：

```golang
	......
	opt = append(opt, keepParam, grpc.UnaryInterceptor(s.interceptor))
	s.server = grpc.NewServer(opt...)
	s.Use(s.recovery(), s.handle(), serverLogging(conf.LogFlag), s.stats(), s.validate())
	s.Use(ratelimiter.New(nil).Limit())
	......
```

其中 `grpc.UnaryInterceptor(...)` 的作用是进行拦截器的初始化，它的原型如下，注意该方法的参数，传入的是 `s.interceptor`：

```go
// UnaryInterceptor returns a ServerOption that sets the UnaryServerInterceptor for the
// server. Only one unary interceptor can be installed. The construction of multiple
// interceptors (e.g., chaining) can be implemented at the caller.
func UnaryInterceptor(i UnaryServerInterceptor) ServerOption {
	return func(o *options) {
		if o.unaryInt != nil {
			panic("The unary server interceptor was already set and may not be reset.")
		}
		o.unaryInt = i
	}
}
```

从 `s.interceptor` 的实现代码，比较清晰的看出来，使用递归的方式将 `s.handlers` 中存储的 interceptor 组织成一个 chain 式逻辑，那么剩下的逻辑就是如何将每个 interceptor 放入这个数组中了。

```golang
// interceptor is a single interceptor out of a chain of many interceptors.
// Execution is done in left-to-right order, including passing of context.
// For example ChainUnaryServer(one, two, three) will execute one before two before three, and three
// will see context changes of one and two.
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
			// 调用最终的 rpc 方法
			return handler(ic, ir)
		}
		i++
		return s.handlers[i](ic, ir, args, chain)
	}

	return s.handlers[0](ctx, req, args, chain)
}
```

`s.handlers` 的定义如下，它就是一个 interceptors 数组：

```golang
// Server is the framework's server side instance, it contains the GrpcServer, interceptor and interceptors.
// Create an instance of Server, by using NewServer().
type Server struct {
	conf  *ServerConfig
	mutex sync.RWMutex

	server   *grpc.Server
	handlers []grpc.UnaryServerInterceptor		//handles 就是一个 UnaryServerInterceptor 数组
}
```

接上面讨论的问题，就是如何向 `s.handlers` 中添加拦截器，`Use` 方法就完成了这个事情，从代码可以看出，每次调用 `Use` 都是向数组的尾部插入新的拦截器：

```golang
// Use attachs a global inteceptor to the server.
// For example, this is the right place for a rate limiter or error management inteceptor.
func (s *Server) Use(handlers ...grpc.UnaryServerInterceptor) *Server {
	finalSize := len(s.handlers) + len(handlers)
	if finalSize >= int(_abortIndex) {
		panic("warden: server use too many handlers")
	}
	mergedHandlers := make([]grpc.UnaryServerInterceptor, finalSize)
	copy(mergedHandlers, s.handlers)
	copy(mergedHandlers[len(s.handlers):], handlers)	// 向尾部插入
	s.handlers = mergedHandlers
	return s
}
```

最后一个问题，让我们再看下 `s.interceptor` 的实现，可以明确知道，构造的链式关系是 `[0]--->[1]--->[2]--->[n-1]-->handler`，这个 handler 就是最终的 RPC 方法，所以 Warden 的拦截器 Chain，** 首先执行的是第 0 号数组位置的拦截器 **：
```golang
	...
	chain = func(ic context.Context, ir interface{}) (interface{}, error) {
		if i == n-1 {
			return handler(ic, ir)
		}
		i++
		return s.handlers[i](ic, ir, args, chain)
	}

	return s.handlers[0](ctx, req, args, chain)
	...
```

至此，对服务端的拦截器 Chain 的分析就完成了。总结下这个整体流程：

* `Warden server` 使用 `Use` 方法进行 `grpc.UnaryServerInterceptor` 的注入，而 `func (s *Server) interceptor` 本身就实现了 `grpc.UnaryServerInterceptor`
* `func (s *Server) interceptor` 可以根据注册的 `grpc.UnaryServerInterceptor` 顺序从前到后依次执行

而 `Warden` 库在初始化的时候将该方法本身注册到了 `gRPC server`，在 `NewServer` 方法内可以看到下面代码：

```go
opt = append(opt, keepParam, grpc.UnaryInterceptor(s.interceptor))
s.server = grpc.NewServer(opt...)
```

如此，完整的服务端拦截器逻辑就串联完成。

####	客户端 Chain
客户端的链逻辑实现和服务端比较类似，这里不再更多详述。

##	0x05	Server 端拦截器运行流程
本小节，描述下拦截器的具体执行过程，需要查看基于 `protobuf` 生成的执行代码：

这个 `_Demo_SayHello_Handler` 方法是关键，该方法会被包装为 `grpc.ServiceDesc` 结构，被注册到 gRPC 内部，具体可在生成的 `pb.go` 代码内查找 `s.RegisterService(&_Demo_serviceDesc, srv)`。

* 当 gRPC server 收到一次请求时，首先根据请求方法从注册到 `server` 内的 `grpc.ServiceDesc` 找到该方法对应的 `Handler` 如：`_Demo_SayHello_Handler` 并执行
* `_Demo_SayHello_Handler` 执行过程请看上面具体代码，当 `interceptor` 不为 `nil` 时，会将 `SayHello` 包装为 `grpc.UnaryHandler` 结构传递给 `interceptor`

这样就完成了 `UnaryServerInterceptor` 的执行过程。

```go
func _Demo_SayHello_Handler(srv interface{}, ctx context.Context, dec func(interface{}) error, interceptor grpc.UnaryServerInterceptor) (interface{}, error) {
	in := new(HelloReq)
	if err := dec(in); err != nil {
		return nil, err
	}
	if interceptor == nil {
		return srv.(DemoServer).SayHello(ctx, in)
	}
	info := &grpc.UnaryServerInfo{
		Server:     srv,
		FullMethod: "/demo.service.v1.Demo/SayHello",
	}
	handler := func(ctx context.Context, req interface{}) (interface{}, error) {
		return srv.(DemoServer).SayHello(ctx, req.(*HelloReq))
	}
	return interceptor(ctx, in, info, handler)
}
```

##	0x06	Client 端拦截器运行流程
同Server端分析类似，先看下基于 `protobuf` 生成的下面代码：

当客户端调用 `SayHello` 时可以看到执行了 `grpc.Invoke` 方法，并且将 `fullMethod` 和其他参数传入，最终会执行下面[代码](https://github.com/grpc/grpc-go/blob/master/call.go)：

```go
func (c *demoClient) SayHello(ctx context.Context, in *HelloReq, opts ...grpc.CallOption) (*google_protobuf1.Empty, error) {
	out := new(google_protobuf1.Empty)
	err := grpc.Invoke(ctx, "/demo.service.v1.Demo/SayHello", in, out, c.cc, opts...)
	if err != nil {
		return nil, err
	}
	return out, nil
}
```
`grpc.Invoke()`方法，如果拦截器（chain）已定义即`cc.dopts.unaryInt != nil`时，则执行自定义的RPC调用：

```golang
// Invoke sends the RPC request on the wire and returns after response is
// received.  This is typically called by generated code.
//
// All errors returned by Invoke are compatible with the status package.
func (cc *ClientConn) Invoke(ctx context.Context, method string, args, reply interface{}, opts ...CallOption) error {
	// allow interceptor to see all applicable call options, which means those
	// configured as defaults from dial option as well as per-call options
	opts = combine(cc.dopts.callOptions, opts)

	if cc.dopts.unaryInt != nil {
		return cc.dopts.unaryInt(ctx, method, args, reply, cc, invoke, opts...)
	}
	return invoke(ctx, method, args, reply, cc, opts...)
}
```

其中的 `unaryInt` 即为客户端连接创建时注册的拦截器，使用下面代码进行注册：
```golang
// WithUnaryInterceptor returns a DialOption that specifies the interceptor for unary RPCs.
func WithUnaryInterceptor(f UnaryClientInterceptor) DialOption {
	return newFuncDialOption(func(o *dialOptions) {
		o.unaryInt = f
	})
}
```

需要注意的是客户端的拦截器在官方 `gRPC` 内也只能支持注册一个，与服务端拦截器 `interceptor chain` 逻辑类似 `warden` 在客户端拦截器也做了相同处理，并且在客户端连接时进行注册。
```golang
// Use attachs a global inteceptor to the Client.
// For example, this is the right place for a circuit breaker or error management inteceptor.
func (c *Client) Use(handlers ...grpc.UnaryClientInterceptor) *Client {
	finalSize := len(c.handlers) + len(handlers)
	if finalSize >= int(_abortIndex) {
		panic("warden: client use too many handlers")
	}
	mergedHandlers := make([]grpc.UnaryClientInterceptor, finalSize)
	copy(mergedHandlers, c.handlers)
	copy(mergedHandlers[len(c.handlers):], handlers)
	c.handlers = mergedHandlers
	return c
}

// chainUnaryClient creates a single interceptor out of a chain of many interceptors.
//
// Execution is done in left-to-right order, including passing of context.
// For example ChainUnaryClient(one, two, three) will execute one before two before three.
func (c *Client) chainUnaryClient() grpc.UnaryClientInterceptor {
	n := len(c.handlers)
	if n == 0 {
		return func(ctx context.Context, method string, req, reply interface{},
			cc *grpc.ClientConn, invoker grpc.UnaryInvoker, opts ...grpc.CallOption) error {
			return invoker(ctx, method, req, reply, cc, opts...)
		}
	}

	return func(ctx context.Context, method string, req, reply interface{},
		cc *grpc.ClientConn, invoker grpc.UnaryInvoker, opts ...grpc.CallOption) error {
		var (
			i            int
			chainHandler grpc.UnaryInvoker
		)
		chainHandler = func(ictx context.Context, imethod string, ireq, ireply interface{}, ic *grpc.ClientConn, iopts ...grpc.CallOption) error {
			if i == n-1 {
				return invoker(ictx, imethod, ireq, ireply, ic, iopts...)
			}
			i++
			return c.handlers[i](ictx, imethod, ireq, ireply, ic, chainHandler, iopts...)
		}

		return c.handlers[0](ctx, method, req, reply, cc, chainHandler, opts...)
	}
}
```

如此完整的客户端拦截器逻辑就串联完成。

##	0x07	拦截器使用
这里我们看下自适应限流拦截器的调用方式，`limiter.Limit()` 是实现了拦截器的 [完整逻辑](https://github.com/go-kratos/kratos/blob/master/pkg/net/rpc/warden/ratelimiter/ratelimiter.go#L48)

```golang
// New new a grpc server.
func New(svc *service.Service) *warden.Server {
	var rc struct {
		Server *warden.ServerConfig
	}
	if err := paladin.Get("grpc.toml").UnmarshalTOML(&rc); err != nil {
		if err != paladin.ErrNotExist {
			panic(err)
		}
	}
	ws := warden.NewServer(rc.Server)

	// 挂载自适应限流拦截器到 warden server，使用默认配置
	limiter := ratelimiter.New(nil)
	ws.Use(limiter.Limit())

	// 注意替换这里：
	// RegisterDemoServer 方法是在 "api" 目录下代码生成的
	// 对应 proto 文件内自定义的 service 名字，请使用正确方法名替换
	pb.RegisterDemoServer(ws.Server(), svc)

	ws, err := ws.Start()
	if err != nil {
		panic(err)
	}
	return ws
}
```

`limiter.Limit()` 的实现：
```golang
// Limit is a server interceptor that detects and rejects overloaded traffic.
func (b *RateLimiter) Limit() grpc.UnaryServerInterceptor {
	return func(ctx context.Context, req interface{}, args *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (resp interface{}, err error) {
		uri := args.FullMethod
		limiter := b.group.Get(uri)
		done, err := limiter.Allow(ctx)
		if err != nil {
			_metricServerBBR.Inc(uri)
			return
		}
		defer func() {
			done(limit.DoneInfo{Op: limit.Success})
			b.printStats(uri, limiter)
		}()
		resp, err = handler(ctx, req)
		return
	}
}
```

##	0x08	Build Your Own 拦截器
[这里摘抄自官方文档] 以服务端拦截器 `serverLogging` 为例，特别要主要的是在 `resp, err := handler(ctx, req)` 前后需要实现哪些逻辑，在此之前，RPC 方法还未执行；在此之后，RPC 方法已经执行完成，可以根据执行结果成功与否来实现自己的逻辑：

```golang
// serverLogging warden grpc logging
func serverLogging() grpc.UnaryServerInterceptor {
	return func(ctx context.Context, req interface{}, info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (interface{}, error) {
        // NOTE: handler 执行之前的拦截代码：主要获取一些关键参数，如耗时计时、ip 等
        // 如果自定义的拦截器只需要在 handler 执行后，那么可以直接执行 handler

		startTime := time.Now()
		caller := metadata.String(ctx, metadata.Caller)
		if caller == "" {
			caller = "no_user"
		}
		var remoteIP string
		if peerInfo, ok := peer.FromContext(ctx); ok {
			remoteIP = peerInfo.Addr.String()
		}
		var quota float64
		if deadline, ok := ctx.Deadline(); ok {
			quota = time.Until(deadline).Seconds()
		}

		// call server handler
		resp, err := handler(ctx, req) // NOTE: 以具体执行的 handler 为分界线！！！

        // NOTE: handler 执行之后的拦截代码：主要进行耗时计算、日志记录
        // 如果自定义的拦截器在 handler 执行后不需要逻辑，这可直接返回

		// after server response
		code := ecode.Cause(err).Code()
		duration := time.Since(startTime)

		// monitor
		statsServer.Timing(caller, int64(duration/time.Millisecond), info.FullMethod)
		statsServer.Incr(caller, info.FullMethod, strconv.Itoa(code))
		logFields := []log.D{
			log.KVString("user", caller),
			log.KVString("ip", remoteIP),
			log.KVString("path", info.FullMethod),
			log.KVInt("ret", code),
			// TODO: it will panic if someone remove String method from protobuf message struct that auto generate from protoc.
			log.KVString("args", req.(fmt.Stringer).String()),
			log.KVFloat64("ts", duration.Seconds()),
			log.KVFloat64("timeout_quota", quota),
			log.KVString("source", "grpc-access-log"),
		}
		if err != nil {
			logFields = append(logFields, log.KV("error", err.Error()), log.KV("stack", fmt.Sprintf("%+v", err)))
		}
		logFn(code, duration)(ctx, logFields...)
		return resp, err
	}
}
```

##	0x09	总结
本文是 Kratos 框架分析的第二篇，主要介绍了 Warden 框架中的拦截器的实现及使用。

##  0x10	参考
-   [Warden 拦截器文档](https://github.com/go-kratos/kratos/blob/master/doc/wiki-cn/warden-mid.md)
-	[Golang: Creating gRPC interceptors](https://davidsbond.github.io/2019/06/14/creating-grpc-interceptors-in-go.html)

转载请注明出处，本文采用 [CC4.0](http://creativecommons.org/licenses/by-nc-nd/4.0/) 协议授权