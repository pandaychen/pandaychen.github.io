---
layout:     post
title:      gRPC的Interceptor实战篇
subtitle:   
date:       2019-11-20
author:     pandaychen
header-img: 
catalog: true
tags:
    - gRPC
    - Interceptor
---

##  背景
在实现 RPC 时，如果想在 RPC 方法的前或后做某些事情，怎么实现？gRPC 提供了拦截器（Interceptor），就可以完成这个功能。比如一个典型的应用场景是，当客户端进行 RPC 请求时，先对请求中的某些字段（如MetaInfo）进行验证，验证通过再执行相应的 RPC 方法。

##  Interceptor的结构体
-   一元拦截器（grpc.UnaryInterceptor）<br>
```go
func UnaryInterceptor(i UnaryServerInterceptor) ServerOption {
    return func(o *options) {
        if o.unaryInt != nil {
            panic("The unary server interceptor was already set and may not be reset.")
        }
        o.unaryInt = i  //设置UnaryServerInterceptor
    }
}
```
其中，要实现一个一元Interceptor，必须实现参数`UnaryServerInterceptor`。`UnaryServerInterceptor`在服务端对于一次 RPC 调用进行拦截。`UnaryServerInterceptor`是一个函数指针，当客户端进行 RPC 调用的时候，首先并不执行用户调用的方法，而是先执行`UnaryServerInterceptor`所指的函数，随后再进入真正要执行的函数。其定义如下：
```go
// UnaryServerInterceptor provides a hook to intercept the execution of a unary RPC on the server. info
// contains all the information of this RPC the interceptor can operate on. And handler is the wrapper
// of the service method implementation. It is the responsibility of the interceptor to invoke handler
// to complete the RPC.
type UnaryServerInterceptor func(
            ctx context.Context,        //请求上下文
            req interface{},            //RPC 方法的请求参数，这里为interface{}
            info *UnaryServerInfo,      //RPC 方法的所有信息(本次调用)
            handler UnaryHandler        //RPC 方法本身(客户端此次实际要调用的函数)
    )(resp interface{}, err error)
```
`UnaryServerInfo`的结构如下：
```go
// UnaryServerInfo consists of various information about a unary RPC on
// server side. All per-rpc information may be mutated by the interceptor.
type UnaryServerInfo struct {
    // Server is the service implementation the user provides. This is read-only.
    //Server是客户编写的服务器端的服务实现，这个成员是只读的
	Server interface{}
    // FullMethod is the full RPC method string, i.e., /package.service/method.
    //FullMethod成员是要调用的方法名，这个方法名interceptor可以进行修改。
	FullMethod string
}
```
所以说，如果需要进行服务端方法调用之前需要实现既定方法验签的话，Interceptor可以这么写：
```go
var interceptor grpc.UnaryServerInterceptor
interceptor = func(ctx context.Context, req interface{}, info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (resp interface{}, err error) {
    fmt.Printf("Before RPC handling. Info: %+v", info)
    /*
        ...
        这里实现我们需要的通用逻辑，如对req中的签名进行验证，成功继续，失败返回error
    */
    //handler是客户端原来打算调用的方法，如果验证成功（在上面的逻辑），执行真正的方法
	resp, err := handler(ctx, req)
	fmt.Printf("After RPC handling. resp: %+v", resp)
    return resp, err
}
```
相应的 gRPC 服务端代码大致如下：
```go
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
适用于gRPC的流式方法，`StreamServerInterceptor` 与 `UnaryServerInterceptor` 使用基本一样。
```go
func StreamInterceptor(i StreamServerInterceptor) ServerOption
type StreamServerInterceptor func(srv interface{}, ss ServerStream, info *StreamServerInfo, handler StreamHandler) error
```

##  Warning
在 gRPC 中，服务器本身只能设置一个`UnaryServerInterceptor`和 `StreamServerInterceptor`。客户端亦是如此，虽然不会报错，但是只有最后一个才起作用。那么如何配置多个拦截器呢？ 下面我们从拦截器的实现代码入手分析下如何实现多个拦截器。
