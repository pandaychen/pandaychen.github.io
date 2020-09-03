---
layout:     post
title:      Kratos 源码分析：CGI 框架 BM （一）
subtitle:   分析基于 gin 改造的框架 blademaster：基础
date:       2020-10-01
header-img: img/super-mario.jpg
author:     pandaychen
catalog:    true
tags:
    - Kratos
---


##  0x00    前言
[blademaster](https://github.com/go-kratos/kratos/tree/master/pkg/net/http/blademaster) 是 Kratos 提供的 HTTP 框架，参考 [gin](https://github.com/gin-gonic/gin) 实现，但剥离了 gin 中的若干代码。其官方介绍是：基于 gin 二次开发，具有快速、灵活的特点，可以方便的开发中间件处理通用或特殊逻辑，基础库默认实现了 log&trace 等。

##  0x01    Web 服务的特点
就笔者而言，一个 CGI 框架可能需要满足如下特性（便于开发上手和功能扩展）：

-   多路由（Multiplexer Router）支持
-   方便封装自己的中间件
-   性能
-   方便嵌入业务逻辑
-   请求参数校验，CGI 签名等框架的安全特性
-   核心指标监控及稳定

##  0x02    blademaster 设计思路
本小节源自 blademaster 的 [官方文档](https://github.com/go-kratos/kratos/blob/master/doc/wiki-cn/blademaster.md)。<br>
在像微服务这样的分布式架构中，经常会有一些需求需要你调用多个服务，但是还需要确保服务的安全性、统一化每次的请求日志或者追踪用户完整的行为等等。要实现这些功能，你可能需要在所有服务中都设置一些相同的属性，虽然这个可以通过一些明确的接入文档来描述或者准入规范来界定，但是这么做的话还是有可能会有一些问题：

1. 你很难让每一个服务都实现上述功能。因为对于开发者而言，他们应当注重的是实现功能。很多项目的开发者经常在一些日常开发中遗漏了这些关键点，经常有人会忘记去打日志或者去记录调用链。但是对于一些大流量的互联网服务而言，一个线上服务一旦发生故障时，即使故障时间很小，其影响面会非常大。一旦有人在关键路径上忘记路记录日志，那么故障的排除成本会非常高，那样会导致影响面进一步扩大。
2. 事实上实现之前叙述的这些功能的成本也非常高。比如说对于鉴权（Identify）这个功能，你要是去一个服务一个服务地去实现，那样的成本也是非常高的。如果说把这个确保认证的责任分担在每个开发者身上，那样其实也会增加大家遗忘或者忽略的概率。

为了解决这样的问题，你可能需要一个框架来帮助你实现这些功能。比如说帮你在一些关键路径的请求上配置必要的鉴权或超时策略。那样服务间的调用会被多层中间件所过滤并检查，确保整体服务的稳定性。

####    blademaster 的设计目标

*   性能优异，不应该掺杂太多业务逻辑的成分
*   方便开发使用，开发对接的成本应该尽可能地小
*   后续鉴权、认证等业务逻辑的模块应该可以通过业务模块的开发接入该框架内
*   默认配置已经是 production ready 的配置，减少开发与线上环境的差异性

####    blademaster 的概览

*   参考 gin 设计整套 HTTP 框架，去除 gin 中不需要的部分逻辑
*   内置一些必要的中间件，便于业务方可以直接上手使用

####    blademaster 架构简介
一般 http 的主要处理流程如下：
![img](https://wx1.sbimg.cn/2020/08/17/3sFae.png)

blademaster 的主要流程也是如此：
![img](https://wx2.sbimg.cn/2020/08/17/3sVgO.png)

从上图来看，blademaster 由几个非常精简的内部模块组成：

*   `Router`：用于根据请求的路径分发请求
*   `Context`：包含了一个完整的请求信息
*   `Handler` 则负责处理传入的 `Context`，`Handlers` 为一个列表，一个串一个地执行

所有的 `middlerware` 均以 `Handler` 的形式存在，这样可以保证 blademaster 自身足够精简且扩展性足够强。

![img](https://wx1.sbimg.cn/2020/08/17/3syp6.png)
上图描述了 blademaster 处理请求的模式及路径，大部分的逻辑都被封装在了各种 `Handler` 中。一般而言，** 业务逻辑作为最后一个 `Handler`**

正常情况下每个 `Handler` 按照顺序一个一个串行地执行下去，但是 `Handler` 中也可以中断整个处理流程，直接输出 `Response`。这种模式常被用于校验登陆、过载保护（如熔断、限流等机制）的 `middleware` 中：一旦发现请求不合法，直接响应拒绝。

请求处理的流程中也可以使用 `Render` 来辅助渲染 `Response`，比如对于不同的请求需要响应不同的数据格式 `JSON`、`XML`，此时可以使用不同的 `Render` 来简化逻辑。


####    代码组织
blademaster 的代码组织及功能简介如下：
*   [server.go](https://github.com/go-kratos/kratos/blob/master/pkg/net/http/blademaster/server.go)：核心 server 结构定义
*   [routergroup.go](https://github.com/go-kratos/kratos/blob/master/pkg/net/http/blademaster/routergroup.go)
*   [tree.go](https://github.com/go-kratos/kratos/blob/master/pkg/net/http/blademaster/tree.go)：CGI 目录树的算法实现
*   [trace.go](https://github.com/go-kratos/kratos/blob/master/pkg/net/http/blademaster/trace.go)：加入链路追踪的逻辑
*   [recovery.go](https://github.com/go-kratos/kratos/blob/master/pkg/net/http/blademaster/recovery.go)：core 处理中间件，简化版
*   [ratelimit.go](https://github.com/go-kratos/kratos/blob/master/pkg/net/http/blademaster/ratelimit.go)：限流器中间件，采用 BBR 限流
*   [prometheus.go](https://github.com/go-kratos/kratos/blob/master/pkg/net/http/blademaster/prometheus.go)：Prometheus-httpserver
*   [perf.go](https://github.com/go-kratos/kratos/blob/master/pkg/net/http/blademaster/perf.go)：性能统计中间件
*   [metrics.go](https://github.com/go-kratos/kratos/blob/master/pkg/net/http/blademaster/metrics.go)：metrics 定义
*   [metadata.go](https://github.com/go-kratos/kratos/blob/master/pkg/net/http/blademaster/metadata.go)：cgi 的元数据定义
*   [logger.go](https://github.com/go-kratos/kratos/blob/master/pkg/net/http/blademaster/logger.go)：日志中间件，记录核心日志
*   [cors.go](https://github.com/go-kratos/kratos/blob/master/pkg/net/http/blademaster/cors.go)：跨域组件

##  0x03    Metrics 统计
blademaster 默认提供了如下几个维度的 metrics[监控采集数据](https://github.com/go-kratos/kratos/blob/master/pkg/net/http/blademaster/metrics.go)：
-   `_metricServerReqDur `：`NewHistogramVec` 类型


##  0x04    核心结构
本篇先从 [server.go](https://github.com/go-kratos/kratos/blob/master/pkg/net/http/blademaster/server.go) 入手，分析下 blademaster 的核心数据结构：



##  0x05    参考
-   [blademaster 官方文档](https://github.com/go-kratos/kratos/blob/master/doc/wiki-cn/blademaster.md)
-   [blademaster](https://github.com/go-kratos/kratos/blob/master/doc/wiki-cn/blademaster-mid.md)
-   [go 语言框架 gin 的中文文档](https://github.com/skyhee/gin-doc-cn)
