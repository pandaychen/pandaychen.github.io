---
layout:     post
title:      gRPC 之 Interceptor 实战篇
subtitle:   如何优雅的实现 gRPC 拦截器链
date:       2019-11-20
author:     pandaychen
header-img:
catalog: true
tags:
    - gRPC
---

##  背景
在实现 RPC 时，如果想在 RPC 方法的前或后做某些事情，怎么实现？gRPC 提供了拦截器（Interceptor），就可以完成这个功能。比如一个典型的应用场景是，当客户端进行 RPC 请求时，先对请求中的某些字段（如 MetaInfo）进行验证，验证通过再执行相应的 RPC 方法。

##  Interceptor 的结构体
-   一元拦截器（grpc.UnaryInterceptor）<br>
包含服务端 `UnaryServerInterceptor` 和客户端 `UnaryClientInterceptor`，这里我们分析 `UnaryServerInterceptor`:
```go
func UnaryInterceptor(i UnaryServerInterceptor) ServerOption {
    return func(o *options) {
        if o.unaryInt != nil {
            panic("The unary server interceptor was already set and may not be reset.")
        }
        o.unaryInt = i  // 设置 UnaryServerInterceptor
    }
}
```
其中，要实现一个一元 Interceptor，必须实现参数 `UnaryServerInterceptor`。`UnaryServerInterceptor` 在服务端对于一次 RPC 调用进行拦截。`UnaryServerInterceptor` 是一个函数指针，当客户端进行 RPC 调用的时候，首先并不执行用户调用的方法，而是先执行 `UnaryServerInterceptor` 所指的函数，随后再进入真正要执行的函数。其定义如下：
```go
// UnaryServerInterceptor provides a hook to intercept the execution of a unary RPC on the server. info
// contains all the information of this RPC the interceptor can operate on. And handler is the wrapper
// of the service method implementation. It is the responsibility of the interceptor to invoke handler
// to complete the RPC.
type UnaryServerInterceptor func(
            ctx context.Context,        // 请求上下文
            req interface{},            //RPC 方法的请求参数，这里为 interface{}
            info *UnaryServerInfo,      //RPC 方法的所有信息 (本次调用)
            handler UnaryHandler        //RPC 方法本身 (客户端此次实际要调用的函数)
    )(resp interface{}, err error)
```
`UnaryServerInfo` 的结构如下：
```go
// UnaryServerInfo consists of various information about a unary RPC on
// server side. All per-rpc information may be mutated by the interceptor.
type UnaryServerInfo struct {
    // Server is the service implementation the user provides. This is read-only.
    //Server 是客户编写的服务器端的服务实现，这个成员是只读的
	Server interface{}
    // FullMethod is the full RPC method string, i.e., /package.service/method.
    //FullMethod 成员是要调用的方法名，这个方法名 interceptor 可以进行修改。
	FullMethod string
}
```
所以说，如果需要进行服务端方法调用之前需要实现既定方法验签的话，Interceptor 可以这么写：
```go
var interceptor grpc.UnaryServerInterceptor
interceptor = func(ctx context.Context, req interface{}, info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (resp interface{}, err error) {
    fmt.Printf("Before RPC handling. Info: %+v", info)
    /*
        ...
        这里实现我们需要的通用逻辑，如对 req 中的签名进行验证，成功继续，失败返回 error
    */
    //handler 是客户端原来打算调用的方法，如果验证成功（在上面的逻辑），执行真正的方法
	resp, err := handler(ctx, req)
	fmt.Printf("After RPC handling. resp: %+v", resp)
    return resp, err
}
```
相应的 gRPC 服务端代码大致如下：
``` golang
listener, err := net.Listen("tcp", fmt.Sprintf(":%s", port))
if err != nil {
    panic(err)
    return
}
var opts []grpc.ServerOption
opts = append(opts, grpc.UnaryInterceptor(interceptor))
server := grpc.NewServer(opts...)
chatprt.RegisterHelloServer()
server.Serve(listener)
```

-   流式拦截器（grpc.StreamInterceptor）
适用于 gRPC 的流式方法，`StreamServerInterceptor` 与 `UnaryServerInterceptor` 使用基本一样。
```go
func StreamInterceptor(i StreamServerInterceptor) ServerOption
type StreamServerInterceptor func(srv interface{}, ss ServerStream, info *StreamServerInfo, handler StreamHandler) error
```

##  Warning
在 gRPC 中，服务器本身只能设置一个 `UnaryServerInterceptor` 和 `StreamServerInterceptor`。客户端亦是如此，虽然不会报错，但是只有最后一个才起作用。那么如何配置多个拦截器呢？ 下面我们从拦截器的实现代码入手分析下如何实现多个拦截器。


##  如何实现多个拦截器
[WithUnaryInterceptor](https://godoc.org/google.golang.org/grpc#WithUnaryInterceptor)
```go
// WithUnaryInterceptor returns a DialOption that specifies the interceptor for
// unary RPCs.
func WithUnaryInterceptor(f UnaryClientInterceptor) DialOption {
	return newFuncDialOption(func(o *dialOptions) {
		o.unaryInt = f
	})
}
```
设想下，如何设计链式的 Interceptor？我脑海中浮现两个方案：
-   回调方式调用
-   数组依次调用

##  已有的方案

### go-grpc-middleware 的 Chain 实现
```go
import "github.com/grpc-ecosystem/go-grpc-middleware"

myServer := grpc.NewServer(
    grpc.StreamInterceptor(grpc_middleware.ChainStreamServer(
        ...
    )),
    grpc.UnaryInterceptor(grpc_middleware.ChainUnaryServer(
       ...
    )),
)
```
看看 `ChainUnaryServer` 方法的实现，在 [chain.go](https://github.com/grpc-ecosystem/go-grpc-middleware/blob/master/chain.go) 的实现：
```go
// ChainUnaryServer creates a single interceptor out of a chain of many interceptors.
//
// Execution is done in left-to-right order, including passing of context.
// For example ChainUnaryServer(one, two, three) will execute one before two before three, and three
// will see context changes of one and two.
func ChainUnaryServer(interceptors ...grpc.UnaryServerInterceptor) grpc.UnaryServerInterceptor {
	n := len(interceptors)

    //return 里面包含了一个完整的回调
	return func(ctx context.Context, req interface{}, info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (interface{}, error) {
        // 定义 chainer 调用
		chainer := func(currentInter grpc.UnaryServerInterceptor, currentHandler grpc.UnaryHandler) grpc.UnaryHandler {
			return func(currentCtx context.Context, currentReq interface{}) (interface{}, error) {
				return currentInter(currentCtx, currentReq, info, currentHandler)
			}
		}

		chainedHandler := handler
		for i := n - 1; i >= 0; i-- {
			chainedHandler = chainer(interceptors[i], chainedHandler)
		}

		return chainedHandler(ctx, req)
	}
}
```
从实现上看，它采用了回调函数调用的方式，调用链为 `chainedHandler[n-1]`-->`chainedHandler[n-2]`-->`...`-->`chainedHandler[0]`-->`handler`

##  参考

转载请注明出处，本文采用 [CC4.0](http://creativecommons.org/licenses/by-nc-nd/4.0/) 协议授权