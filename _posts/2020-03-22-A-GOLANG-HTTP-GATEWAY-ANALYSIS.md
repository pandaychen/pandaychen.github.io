---
layout:     post
title:      一个 Http(s) 网关的实现分析
subtitle:   如何使用几百行代码实现一个高可用的反向代理服务
date:       2020-03-22
author:     pandaychen
header-img:
catalog: true
category:   false
tags:
    - 反向代理
---


##  0x00    前言
本篇文章主要是对先前阅读过一篇关于 http-gateway 实现的文章的总结：[Let's Create a Simple Load Balancer With Go](https://kasvith.me/posts/lets-create-a-simple-lb-go/) 的小结。

##  0x01    目标拆解
我们从要实现的功能及目标出发，来拆解一个 **高可用** 的（反向代理）网关，需要支持哪些功能？

![image](https://wx2.sbimg.cn/2020/05/22/http-lb1.png)

从架构图来看

-   健康检查：如果其中的一个后端发生故障该怎么办？我们当然不希望把流量定向给它。我们只能把流量路由给正常运行的服务。


##  参考
-   [Let's Create a Simple Load Balancer With Go](https://kasvith.me/posts/lets-create-a-simple-lb-go/)
-   [Gobetween - Introduction](http://gobetween.io/documentation.html)
-   [Yet another load balancer](https://github.com/onestraw/golb)
-   [Vulcand Gateway](https://vulcand.github.io/quickstart.html)