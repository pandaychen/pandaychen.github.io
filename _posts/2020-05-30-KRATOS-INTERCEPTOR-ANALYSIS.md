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

##	0x04	Warden拦截器链（Chain）
为了减轻开发者对拦截器的依赖，gRPC特意要求，无论服务端or客户端只能注册一个拦截器（官方的说法：Only one unary interceptor can be installed. The construction of multiple interceptors (e.g., chaining) can be implemented at the caller.），但是实际中，一个Interceptor肯定是不够的，所以需要对单拦截器进行扩展，那就是拦截器链。Warden包针对服务端和客户端都封装了Chain的实现。

####	服务端Chain
对于服务端，Warden封装的代码[在此](https://github.com/go-kratos/kratos/blob/master/pkg/net/rpc/warden/server.go#L263)

在服务端注册时，调用`opt = append(opt, keepParam, grpc.UnaryInterceptor(s.interceptor))`，告诉gRPC，拦截器Chain该如何调用，调用的代码如下：

```golang
	......
	opt = append(opt, keepParam, grpc.UnaryInterceptor(s.interceptor))
	s.server = grpc.NewServer(opt...)
	s.Use(s.recovery(), s.handle(), serverLogging(conf.LogFlag), s.stats(), s.validate())
	s.Use(ratelimiter.New(nil).Limit())
	......
```

其中`grpc.UnaryInterceptor(...)`的作用是进行拦截器的初始化，它的原型如下，注意该方法的参数，传入的是`s.interceptor`：

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

从`s.interceptor`的实现代码，比较清晰的看出来，使用递归的方式将`s.handlers`中存储的interceptor组织成一个chain式逻辑，那么剩下的逻辑就是如何将每个interceptor放入这个数组中了。

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
			//调用最终的rpc方法
			return handler(ic, ir)
		}
		i++
		return s.handlers[i](ic, ir, args, chain)
	}

	return s.handlers[0](ctx, req, args, chain)
}
```

`s.handlers`的定义如下，它就是一个interceptors数组：

```golang
// Server is the framework's server side instance, it contains the GrpcServer, interceptor and interceptors.
// Create an instance of Server, by using NewServer().
type Server struct {
	conf  *ServerConfig
	mutex sync.RWMutex

	server   *grpc.Server
	handlers []grpc.UnaryServerInterceptor		//handles就是一个UnaryServerInterceptor数组
}
```

接上面讨论的问题，就是如何向`s.handlers`中添加拦截器，`Use`方法就完成了这个事情，从代码可以看出，每次调用`Use`都是向数组的尾部插入新的拦截器：

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
	copy(mergedHandlers[len(s.handlers):], handlers)	//向尾部插入
	s.handlers = mergedHandlers
	return s
}
```

最后一个问题，让我们再看下`s.interceptor`的实现，可以明确知道，构造的链式关系是`[0]--->[1]--->[2]--->[n-1]-->handler`，这个handler就是最终的RPC方法，所以Warden的拦截器Chain，**首先执行的是第0号数组位置的拦截器**：
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

至此，对服务端的拦截器Chain的分析就完成了。


-	[客户端Chain]



##	0x04	Server端拦截器运行流程


##	总结


##  参考
-   [Warden 拦截器文档](https://github.com/go-kratos/kratos/blob/master/doc/wiki-cn/warden-mid.md)
-	[Golang: Creating gRPC interceptors](https://davidsbond.github.io/2019/06/14/creating-grpc-interceptors-in-go.html)

转载请注明出处，本文采用 [CC4.0](http://creativecommons.org/licenses/by-nc-nd/4.0/) 协议授权