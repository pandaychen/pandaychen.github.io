---
layout:     post
title:      开源的gRPC Interceptor汇总 
subtitle:
date:       2020-02-25
author:     pandaychen
header-img:
catalog: true
category:   false
tags:
    - Interceptor
---

##  0x00    前言
&emsp;&emsp;前面文章，讨论了Interceptor的实现及使用。Interceptor极大的丰富了gRPC的功能。注意，服务器只能配置一个 unary interceptor和 stream interceptor，否则会报错，客户端也是，虽然不会报错，但是只有最后一个才起作用。 如果你想配置多个，可以使用这个[chain.go](https://github.com/grpc-ecosystem/go-grpc-middleware/blob/master/chain.go)。

##  0x01    开源的Interceptor
[grpc-ecosystem](https://github.com/grpc-ecosystem/go-grpc-middleware)中包含了不少优秀的Interceptor，这里列举一些笔者使用过的：

##  0x03    使用开源的Interceptor的心得
*   查看拦截器的功能
*   拦截器链中的先后顺序（摆放）
*   改造，融合到自己的框架中


##  0x04    参考
-   [gRPC的那些事 - interceptor](https://colobu.com/2017/04/17/dive-into-gRPC-interceptor/)