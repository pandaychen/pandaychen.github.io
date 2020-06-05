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
客户端拦截器 `grpc.UnaryClientInterceptor` 的声明 [在此](https://github.com/grpc/grpc-go/blob/master/interceptor.go)，和 `UnaryServerInterceptor` 类似，只不过，服务端的 RPC 方法叫 `handler`，客户端叫 `invoker`：

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


##	总结


##  参考
-   [Warden 拦截器文档](https://github.com/go-kratos/kratos/blob/master/doc/wiki-cn/warden-mid.md)
-	[Golang: Creating gRPC interceptors](https://davidsbond.github.io/2019/06/14/creating-grpc-interceptors-in-go.html)

转载请注明出处，本文采用 [CC4.0](http://creativecommons.org/licenses/by-nc-nd/4.0/) 协议授权