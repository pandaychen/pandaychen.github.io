---
layout:     post
title:      Consul 服务治理的那些事（二）
subtitle:   Consul 的细节梳理
date:       2020-10-13
author:     pandaychen
catalog:    true
tags:
    - Consul
    - 负载均衡
    - gRPC
---

##  0x00    前言
上一篇文章 [Consul 服务治理的那些事](https://pandaychen.github.io/2019/10/12/CONSUL-APPLICATION/) 介绍了 Consul 的基础原理和使用，本文对 Consul 构建项目中的一些细节再做下梳理。

关于 Consul 的一些高级方法，推荐 [玩转 CONSUL 系列的文章](http://vearne.cc/archives/13983)

##  0x01    服务注册与发现
Consul 的服务发现模式如下图所示：

-   服务注册: 服务端将服务的信息注册到 Consul 里
-   服务发现: 客户端从 Consul 里发现服务信息，主要是服务的地址
-   健康检查: Consul 负责检查服务器的健康状态，基于设置检查策略不通过的情况下剔除节点

![img-consul-service]()

####   Watch 机制：阻塞查询
同 Etcd 的 Watch 一样，Consul 也提供了 Watch 机制，基于 HTTP Long Polling 机制来实现。

##  0x02    Consul 提供的 API



##  0x03    Consul 集群监控指标
-   进程：CPU、内存、goroutine、连接数
-   raft：成员状态变动、提交速率、提交耗时、同步心跳、同步延时
-   RPC：连接数、跨 DC 请求数
-   写负载：注册及解注册速率
-   读负载：`Catalog`/`Health`/`PreparedQuery` 请求量，执行耗时

##  参考
-   [记一次 Consul 故障分析与优化](https://www.infoq.cn/article/qv02j2ezmjbow8ckcopg)
-   [golang consul-grpc 服务注册与发现](http://www.hatlonely.com/2018/06/23/golang-consul-grpc-%E6%9C%8D%E5%8A%A1%E6%B3%A8%E5%86%8C%E4%B8%8E%E5%8F%91%E7%8E%B0/index.html)

转载请注明出处，本文采用 [CC4.0](http://creativecommons.org/licenses/by-nc-nd/4.0/) 协议授权


