---
layout:     post
title:      Envoy 应用入门（TODO）
subtitle:
date:       2023-10-03
author:     pandaychen
catalog:    true
tags:
    - Envoy
---


##  0x00    前言
Envoy 是一个开源的边缘服务代理，也是 Istio Service Mesh 默认的数据平面，专为云原生应用程序设计。与 HAProxy 以及 Nginx 等传统 Proxy 依赖静态配置文件来定义各种资源以及数据转发规则不同，Envoy 几乎所有配置都可以通过订阅来动态获取。对应的发现服务以及各种各样的 API 统称为 `xDS`。Envoy 与 `xDS` 之间通过 Proto 约定请求和响应的数据模型，不同类型资源，对应的数据模型也不同。


##  第一个例子

配置文件[参考](https://github.com/pandaychen/envoy-study/blob/main/basic/envoy-1.yaml)


```BASH
docker logs -f  --tail=200 cfb4ba22c969
```


##  第三个例子：使用 SSL/TLS 保护流量


##  Nginx 迁移到 Envoy


##  0x 参考
-   [Envoy 架构及其在网易轻舟的落地实践](https://mp.weixin.qq.com/s?src=11&timestamp=1701179881&ver=4924&signature=r4fQN4Dk3Zuce3C1ojExLYSF-G0a1j20ogEqbD7ZACIiyi8LCYh0ZtwImTjC7hWkqTo2BtMQ69aLJbH7Mwkwj297AgS2cmLSN6OzeCn5Kjbz2JtZKRva6wIY3AuDfiKF&new=1)
-   [Envoy 的架构与基本配置解析](https://jimmysong.io/blog/envoy-archiecture-and-terminology/)
-   [](https://academy.tetrate.io/courses/envoy-fundamentals)
-   [Envoy 架构理解 -- 理解 xDS/Listener/Cluster/Router/Filter](https://blog.csdn.net/gengzhikui1992/article/details/117449972)
-   [Envoy 入门教程](https://www.qikqiak.com/envoy-book/)
-   [envoy proxy 调研笔记](https://hxysayhi.com/blog/posts/4a1349e1/)
-   [动态转发代理](https://cloudnative.to/envoy/configuration/http/http_filters/dynamic_forward_proxy_filter.html)
-   [Envoy 简单入门示例](https://www.qikqiak.com/post/envoy-usage-demo/)