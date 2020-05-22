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

![image](https://wx2.sbimg.cn/2020/05/22/http-lb1.png)

##  0x01    目标拆解
我们从要实现的功能及目标出发，来拆解一个 **高可用** 的（反向代理）网关，需要支持哪些功能？

![image](https://wx2.sbimg.cn/2020/05/22/http-lb-all-part.png)

从架构图来看，我们将网关划分为控制平面（control plane）和数据平面（data plane）：

控制平面包括：
1.  Backend-Node-Manage：负责维护后端（Backend）的增删查改
2.  Backend-Node-Discovery：后端的服务发现模块
3.  Backend-Node-HealthyCheck：对后端的健康检查，如果后端有故障，要及时剔除
4.  Backend-Node-LB-picker：选择何种负载均衡算法将客户端请求代理到后端
5.  Client-API-interface：客户端调用的API

数据平面只负责转发数据流，即类似反向代理的功能，不过也可以实现中间人（MITM）的功能



##  参考
-   [Let's Create a Simple Load Balancer With Go](https://kasvith.me/posts/lets-create-a-simple-lb-go/)
-   [Gobetween - Introduction](http://gobetween.io/documentation.html)
-   [Yet another load balancer](https://github.com/onestraw/golb)
-   [Vulcand Gateway Doc](https://vulcand.github.io/quickstart.html)
-   [Vulcand - Programmatic load balancer backed by Etcd](https://github.com/vulcand/vulcand)