---
layout:     post
title:      Google SRE 弹性熔断算法实现分析（未完待续）
subtitle:
date:       2020-05-10
author:     pandaychen
header-img: img/golang-tools-fun.png
catalog: true
category:   false
tags:
    - Ratelimit
---

##  0x00    前言
&emsp;&emsp; 这篇文章，了解下 Google SRE 中的过载保护的处理机制 --[Handling Overload](https://landing.google.com/sre/sre-book/chapters/handling-overload)

之前的文章，分析了 Hystrix-Go 熔断算法的实现，其算法核心是：当请求失败比率达到一定阈值之后，熔断器开启，并休眠一段时间（由配置决定），这段休眠期过后，熔断器将处于半开状态，在此状态下将试探性的放过一部分流量，如果这部分流量调用成功后，再次将熔断器关闭，否则熔断器继续保持开启并进入下一轮休眠周期。

但这个熔断算法有一个问题，过于一刀切。是否可以做到在熔断器开启状态下（但是后端未 Shutdown）仍然可以放行少部分流量呢？当然，这里有个前提，需要看后端此时还能够接受多少流量。下一步我们来看看 Google 的实现。

##  0x01    Google 的做法

We implemented client-side throttling through a technique we call adaptive throttling：该算法称为自适应限流。

该算法统计的指标依赖如下两种：
-   requests
The number of requests attempted by the application layer(at the client, on top of the adaptive throttling system)

-   accepts
The number of requests accepted by the backend


##  参考
-   [Handling Overload](https://landing.google.com/sre/sre-book/chapters/handling-overload/#eq2101)