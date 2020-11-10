---
layout:     post
title:      gRPC Interceptor：go-grpc-middleware 介绍与使用
subtitle:   优秀的 gRPC 开源中间件实现：go-grpc-middleware
date:       2020-02-25
author:     pandaychen
header-img:
catalog: true
category:   false
tags:
    -   gRPC
    -   限流
---

##  0x00    前言
&emsp;&emsp; Interceptor 机制极大的扩展了 gRPC 的功能。注意，服务器只能配置一个 Unary interceptor 和 Stream interceptor，否则会报错。客户端也类似，虽然不会报错，但是只有最后一个才起作用。 如果你想配置多个，可以使用这个 [chain.go](https://github.com/grpc-ecosystem/go-grpc-middleware/blob/master/chain.go)。

##  0x01    开源的 Interceptor
&emsp;&emsp; 本文介绍的开源项目 [grpc-ecosystem：go-grpc-middleware](https://github.com/grpc-ecosystem/go-grpc-middleware)，提供了拦截器 Interceptor 链式的功能，即可以将多个拦截器组合成一个拦截器链。此外，该库还提供了常用拦截器的实现（可使用 chain 方式将它们打包起来），从官方文档看，有如下这些，涵盖了认证、日志、监控、客户端重连及服务端（参数校验 / recovery 恢复 / 限流等）：

#### Auth
   * [`grpc_auth`](auth) - a customizable (via `AuthFunc`) piece of auth middleware

#### Logging
   * [`grpc_ctxtags`](tags/) - a library that adds a `Tag` map to context, with data populated from request body
   * [`grpc_zap`](logging/zap/) - integration of [zap](https://github.com/uber-go/zap) logging library into gRPC handlers.
   * [`grpc_logrus`](logging/logrus/) - integration of [logrus](https://github.com/sirupsen/logrus) logging library into gRPC handlers.
   * [`grpc_kit`](logging/kit/) - integration of [go-kit](https://github.com/go-kit/kit/tree/master/log) logging library into gRPC handlers.

#### Monitoring
   * [`grpc_prometheus`](https://github.com/grpc-ecosystem/go-grpc-prometheus) - Prometheus client-side and server-side monitoring middleware
   * [`otgrpc`](https://github.com/grpc-ecosystem/grpc-opentracing/tree/master/go/otgrpc) - [OpenTracing](http://opentracing.io/) client-side and server-side interceptors
   * [`grpc_opentracing`](tracing/opentracing) - [OpenTracing](http://opentracing.io/) client-side and server-side interceptors with support for streaming and handler-returned tags

#### Client
   * [`grpc_retry`](retry/) - a generic gRPC response code retry mechanism, client-side middleware

#### Server
   * [`grpc_validator`](validator/) - codegen inbound message validation from `.proto` options
   * [`grpc_recovery`](recovery/) - turn panics into gRPC errors
   * [`ratelimit`](ratelimit/) - grpc rate limiting by your own limiter

如果日常使用 gRPC 完成服务端项目需要使用拦截器的话，直接从现有的轮子里面寻找就可以了:)

##  0x02    chain 实现
go-grpc-middleware 的 [拦截器链](https://github.com/grpc-ecosystem/go-grpc-middleware/blob/master/chain.go) 实现。

##  0x03    go-grpc-middleware 使用
本小节列举一些拦截器的使用例子：

####    限流 Ratelimit
先简单看下限流拦截器的 [实现](https://github.com/grpc-ecosystem/go-grpc-middleware/blob/master/ratelimit/ratelimit.go)：

需要开发者实现 `Limiter` 接口，此接口中包含了唯一的方法 `Limit() bool`，也很好理解，返回 `true` 表示需要被拦截（限流生效），反之请求被 Pass。可以结合常见的限流方案使用。

PS：这里可以根据限流返回的结果做一些 [额外的事情](https://github.com/go-kratos/kratos/blob/master/pkg/net/rpc/warden/ratelimiter/ratelimiter.go#L48)，参考 Kratos 项目的 [BBR 限流方案](https://github.com/go-kratos/kratos/blob/master/docs/ratelimit.md)，是一款自适应限流拦截器的实现。参见 [Kratos 源码分析：限流器 Limiter](https://pandaychen.github.io/2020/07/12/KRATOS-LIMITER/) 一文分析。

```golang
// Limiter defines the interface to perform request rate limiting.
// If Limit function return true, the request will be rejected.
// Otherwise, the request will pass.
type Limiter interface {
	Limit() bool
}

// UnaryServerInterceptor returns a new unary server interceptors that performs request rate limiting.
func UnaryServerInterceptor(limiter Limiter) grpc.UnaryServerInterceptor {
	return func(ctx context.Context, req interface{}, info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (interface{}, error) {
      // 限流
		if limiter.Limit() {
			return nil, status.Errorf(codes.ResourceExhausted, "%s is rejected by grpc_ratelimit middleware, please retry later.", info.FullMethod)
        }
      // 不限流，进入下面的拦截器逻辑
		return handler(ctx, req)
	}
}
```

##  0x04    使用开源的 Interceptor 的心得
*   明确使用时客户端拦截器还是服务端拦截器
*   查看拦截器的功能
*   拦截器链中的先后顺序（摆放顺序），非常重要，比如一般 `recovery` 拦截器通常建议放置在首位
*   改造，融合到自己的框架中

##  0x05    参考
-   [gRPC 的那些事 - interceptor](https://colobu.com/2017/04/17/dive-into-gRPC-interceptor/)
-   [go-grpc-interceptor](https://github.com/mercari/go-grpc-interceptor)

转载请注明出处，本文采用 [CC4.0](http://creativecommons.org/licenses/by-nc-nd/4.0/) 协议授权