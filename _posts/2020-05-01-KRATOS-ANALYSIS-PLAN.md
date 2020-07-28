---
layout:     post
title:      Kratos 源码分析：计划
subtitle:   分析微服务开发框架 Kratos
date:       2020-05-01
author:     pandaychen
catalog:    true
tags:
    - Kratos
---

##  计划

计划在工作之余研究下 [Kratos 项目](https://github.com/go-kratos/kratos) 的源码及实现，包含了微服务发现、gRPC 封装、Opentracing、Monitoring、CGI 组件等等。

以下排名不分先后

-   Lazy Load Container
-   gRPC-Warden 拦截器（链）及实现
-   gRPC-Warden 中的超时传递
-   gRPC-Warden 中的多消费者订阅 - Watcher 模式（gRPC-Resolver）
-   gRPC-Warden 的服务端封装
-   gRPC-Warden 的客户端封装及使用
-   gRPC-Warden 的服务端封装及使用
-   gRPC-Warden 的限流器
-   gRPC-Warden 的熔断器
-   gRPC-Warden 的负载均衡算法 P2C 分析
-   gRPC-Warden 的负载均衡算法 WRR 分析
-   gRPC-Warden 的若干细节（补充）
-   Kratos 中的 Naming 机制分析
-   Kratos 中的 Monitoring 与 Metrics
-   Kratos 中的 Opentracing 分析与应用
-   Kratos 中的 Metadata 元数据


转载请注明出处，本文采用 [CC4.0](http://creativecommons.org/licenses/by-nc-nd/4.0/) 协议授权