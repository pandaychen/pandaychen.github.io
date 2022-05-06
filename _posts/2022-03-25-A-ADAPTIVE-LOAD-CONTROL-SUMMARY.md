---
layout: post
title: 微服务项目中的自适应技术（adaptive）分析与应用（TODO）
subtitle: 分析 go-zero 框架中自适应技术的运用
date: 2022-03-25
header-img: img/super-mario.jpg
author: pandaychen
catalog: true
tags:
  - 微服务框架
  - 自适应技术
---

## 0x00 前言
本篇文章分析下自适应技术在微服务领域的实践，以 go-zero[项目](https://go-zero.dev/) 为例，此项目非常值得借鉴。

##  0x01 背景
自适应解决了什么问题呢？以 circuitbreaker[熔断器](https://resilience4j.readme.io/docs/circuitbreaker) 而言，可选的配置参数非常多，和系统预期吞吐 / qps 的经验值都会有关系，配置合适的参数是一件很麻烦的事情。所以，通过自适应算法能让我们尽量少关注参数，只要简单配置就能满足大部分场景。

## 0x02   自适应的负载均衡实现
自适应的负载均衡，意为自动的选择指标最优的节点进行请求（如负载低、延时低等），负载低考虑 CPU / 内存，延时主要指接口响应；此外，要求可以动态发现后端节点以及隔离故障节点。从代码 [实现]() 看，go-zero 采用了 P2C 算法 + EWMA 来实现自适应的 LB 机制。

####  基础
- P2C：在多个节点中随机选择 2 个，然后再此 2 中选择一个最优
- EWMA： 指数移动加权平均法，其意义在于只需要保存最近一次的数值（如接口延时），利用加权因子来预估时间区间的平均值，算法的具体含义可参考[此文]()



##  0x03  自适应的熔断器实现


##  0x04  自适应的限流器实现


## 0x05 参考
-	[go-zero 的 P2C 算法实现](https://github.com/zeromicro/go-zero/blob/master/zrpc/internal/balancer/p2c/p2c.go)
- [自适应负载均衡算法原理与实现](https://learnku.com/articles/60059)