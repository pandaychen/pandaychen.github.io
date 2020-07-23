---
layout:     post
title:      Kratos 源码分析：Warden 之 gRPC-Client 封装
subtitle:   分析 Warden 的 Client 端封装
date:       2020-07-20
author:     pandaychen
catalog:    true
tags:
    - Kratos
---

##  0x00    前言
Kratos 的 Warden 框架 [client.go](https://github.com/go-kratos/kratos/blob/master/pkg/net/rpc/warden/client.go) 封装了 gRPC 的客户端启动的核心逻辑。
客户端封装的重点在下面几个方面：
-   客户端启动 & 配置流程
-   客户端拦截器链的 "安装" 顺序
-   `tracer`、`metrics` 以及 `breaker` 与 `grpc.Client` 的结合
-	客户端的 `Naming` 和 gRPC 框架结合
-   客户端调用的 LoadBalance 算法

上面列举的条目中，注意，Warden 的 Server 端默认开启了 `limiter`、而客户端默认开启的是 `breaker`、当然也可以加入 `limiter`

##  0x01    Client 端分析
`warden.Client` 结构如下，默认封装了熔断器 `breaker *breaker.Group` 及拦截器数组 `handlers []grpc.UnaryClientInterceptor`：
```golang
// Client is the framework's client side instance, it contains the ctx, opt and interceptors.
// Create an instance of Client, by using NewClient().
type Client struct {
	conf    *ClientConfig
	breaker *breaker.Group
	mutex   sync.RWMutex

	opts     []grpc.DialOption
	handlers []grpc.UnaryClientInterceptor
}
```

####    客户端初始化
Warden 提供了外部调用的方法 `NewClient`，用来创建 `warden.Client` 结构，注意默认开启的 LB 算法是 `P2C`：
```golang
// NewClient returns a new blank Client instance with a default client interceptor.
// opt can be used to add grpc dial options.
func NewClient(conf *ClientConfig, opt ...grpc.DialOption) *Client {
	c := new(Client)
	if err := c.SetConfig(conf); err != nil {
		panic(err)
	}
	c.UseOpt(grpc.WithBalancerName(p2c.Name))
	c.UseOpt(opt...)
	return c
}
```

##  参考
-   [client.go](https://github.com/go-kratos/kratos/blob/master/pkg/net/rpc/warden/client.go)