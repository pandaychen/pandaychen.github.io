
---
layout:     post
title:      微服务基础之熔断保护（Breaker）
subtitle:
date:       2020-04-01
author:     pandaychen
header-img:
catalog: true
category:   false
tags:
    - 熔断
---


##  0x00    前言

什么是熔断器/Breaker？
熔断器是为了当依赖的服务已经出现故障时，主动阻止对依赖服务的请求（通常情况下是执行本地的降级办法，亦或直接返回错误），从而保证自身服务的正常运行不受依赖服务影响，防止雪崩效应。

一般而言，熔断保护策略部署在服务端保护策略的最后一级，在限流之后。

##  0x01	原理

##  0x03	总结

##  0x04	参考
-   [微服务-熔断机制](http://blog.zhuxingsheng.com/blog/micro-service-fuse-mechanism.html)
-   [CircuitBreaker](https://martinfowler.com/bliki/CircuitBreaker.html)

转载请注明出处，本文采用 [CC4.0](http://creativecommons.org/licenses/by-nc-nd/4.0/) 协议授权