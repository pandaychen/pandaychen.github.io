---
layout:     post
title:      开源的 gRPC Interceptor 汇总 
subtitle:
date:       2020-02-25
author:     pandaychen
header-img:
catalog: true
category:   false
tags:
    - gRPC
---

##  0x00    前言
&emsp;&emsp; 前面文章，讨论了 Interceptor 的实现及使用。Interceptor 极大的丰富了 gRPC 的功能。注意，服务器只能配置一个 Unary interceptor 和 Stream interceptor，否则会报错，客户端也是，虽然不会报错，但是只有最后一个才起作用。 如果你想配置多个，可以使用这个 [chain.go](https://github.com/grpc-ecosystem/go-grpc-middleware/blob/master/chain.go)。

##  0x01    开源的 Interceptor
[grpc-ecosystem](https://github.com/grpc-ecosystem/go-grpc-middleware) 中包含了不少优秀的 Interceptor，这里列举一些笔者使用过的：

##  0x03    使用开源的 Interceptor 的心得
*   明确使用时客户端拦截器还是服务端
*   查看拦截器的功能
*   拦截器链中的先后顺序（摆放）
*   改造，融合到自己的框架中


##  0x04    参考
-   [gRPC 的那些事 - interceptor](https://colobu.com/2017/04/17/dive-into-gRPC-interceptor/)
-   [go-grpc-interceptor](https://github.com/mercari/go-grpc-interceptor)

转载请注明出处，本文采用 [CC4.0](http://creativecommons.org/licenses/by-nc-nd/4.0/) 协议授权