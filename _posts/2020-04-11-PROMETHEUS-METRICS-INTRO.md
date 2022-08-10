---
layout: post
title: 理解 Prometheus 的基本数据类型及应用（基础篇）
subtitle: Prometheus 介绍及基础应用
date: 2020-04-11
author: pandaychen
header-img: golang-horse-fly.png
catalog: true
category: false
tags:
  - Prometheus
  - Metrics
---

## 0x00 前言
本文分析下 Prometheus 以及常用的 Metrics，Metrics 是一种采样数据的总称，是一种对于度量计算单位的抽象。Prometheus 是一个数据监控的解决方案，能随时掌握系统运行的状态，快速定位问题和排除故障。并提供了从指标暴露，到指标抓取、存储和可视化，以及最后的监控告警等一系列组件。

prometheus 的整体架构如下：
![architecture](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/metrics/prometheus/architecture.png)

##  0x01  Prometheus 介绍

####  指标暴露
对应于监控系统的两种方式，针对服务端而言：
-  被动方式（PUSH）：适用于短时间执行的脚本任务或者不好直接 Pull 指标的服务（进程），Prometheus 提供了 PushGateWay 网关给此类任务将服务指标主动推 Push 到网关，Prometheus 再从网关里 Pull 指标
-  主动方式（PULL）：使用 Prometheus 提供的官方的 SDK，利用 SDK 可以自定义并导出自己的业务指标（或者使用 Exporter）

一般来说，主动模式对 Server 端压力较大，被动模式对 Server 端压力相对会小。一般考虑如下选择：
-  PULL 方式拉取：Server 与 Node 之间网络直连（或者自己实现一个代理，让 Prometheus 从代理进行 PULL）
-  Promethues Server 无法直接和 Node 节点通讯的情况下，可以通过 PUSH 搭配 PushGateWay 的方式


PULL 模型：监控服务主动拉取被监控服务的指标，被监控服务一般通过主动暴露 metrics 端口或者通过 Exporter 的方式暴露指标，监控服务依赖服务发现模块发现被监控服务，从而去定期的抓取指标
![PULL](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/metrics/prometheus/pull-based.png)

PUSH 模型：被监控服务主动将指标推送到监控服务，可能需要对指标做协议适配，必须得符合监控服务要求的指标格式
![PUSH](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/metrics/prometheus/push-based.png)


####  指标存储和查询
-  指标抓取后会存储在Prometheus内置的时序数据库
-  PromQL 查询语言给我们做指标的查询
-  可以在 Prometheus 的 WebUI 上通过 PromQL，可视化查询我们的指标，也可以很方便的接入第三方的可视化工具 grafana 进行查询


####  工作原理
Prometheus的从被监控服务的注册到指标抓取到指标查询的流程如下：
![worker](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/metrics/prometheus/worker-flow.png)

1、 （被监控）服务注册<br>
被监控服务在 Prometheus 中是一个 Job 存在，被监控服务的所有实例在 Prometheus 中是一个 target 的存在，所以被监控服务的注册就是在 Prometheus 中注册一个 Job 和其所有的 target。注册有如下两种类型：

-  静态注册：配置文件配置采集指标的IP+端口，配合Prometheus提供的`reload` API以及`--web.enable-lifecycle`参数实现热更新
-  动态注册：配置文件配置endpoint地址，类似于服务发现（支持 Consul/DNS/文件/K8S 等多种服务发现机制）

如，静态注册，启动指令`prometheus --config.file=/usr/local/etc/prometheus.yml --web.enable-lifecycle`：
```yaml
scrape_configs:
  - job_name: "prometheus"
    static_configs:
    - targets: ["localhost:9090"]
```

动态注册，使用consul，consul 地址`localhost:8500`，服务名是 `node_exporter`，在该服务下有一个 Exporter 实例`localhost:9600`
```yaml
- job_name: "node_export_consul"
    metrics_path: /node_metrics
    scheme: http
    consul_sd_configs:
      - server: localhost:8500
        services:
          - node_exporter
```

2、指标抓取和存储<br>
Prometheus 对指标的抓取采取主动 PULL 方式，即周期性的请求被监控服务暴露的 metrics 接口或者是 PushGateway，从而获取到 Metrics 指标，默认时间是 `15s` 抓取一次；抓取到的指标会被以时间序列的形式保存在内存中，并且定时刷到磁盘上（默认`2h`）

####  0x02  基础：Metrics

#### 数据模型
Prometheus 采集的所有指标都是以时间序列的形式进行存储，每一个时间序列有三部分组成：

![struct](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/metrics/prometheus/metrics-basic-data-structure.png)

-  指标名和指标标签集合：metric_name{<label1=v1>,<label2=v2>....}
   -  指标名：表示这个指标是监控哪一方面的状态，如 `http_request_total` 表示HTTP请求数量
   -  指标标签，描述这个指标有哪些维度，比如 `http_request_total` 指标，有请求状态码 `code = 200/400/500`，请求方式：`method = GET/POST` 等标签
-  时间戳：描述当前时间序列的时间（ms）
-  样本值：当前监控指标的具体数值，比如 `http_request_total` 的值就代表请求数是多少

####  指标的格式
格式如下：

```text
# HELP    // HELP：这里描述的指标的信息，表示这个是一个什么指标，统计什么的
# TYPE    // TYPE：这个指标是什么类型的
<metric name>{<label name>=<label value>, ...}  value    // 指标的具体格式，< 指标名 >{标签集合} 指标值
```

时间序列是时间和标签所发构成的多维度内数据，样本为实际的时间序列，每个序列包括一个`float64` 的值和一个毫秒级的时间戳。下图为样本的示例：

![sanple1](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/metrics/prometheus/sample-1.png)


####  指标类型
Prometheus 底层存储上其实并没有对指标做类型的区分，都是以时间序列的形式存储，但是为了方便用户的使用和理解不同监控指标之间的差异，Prometheus 定义计数器 counter/仪表盘 gauge/直方图 histogram以及摘要 summary这四种Metrics类型。

**Gauge/Counter是数值指标，代表数据的变化情况，Histogram/Summary是统计类型的指标，表示数据的分布情况**

![four](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/metrics/four-metrics-type.png)

下面分别介绍指标类型，图来源于[一文带你了解 Prometheus](https://cloud.tencent.com/developer/article/1999843)。

#### Gauges
Gauges 理解为（待监控的）瞬时状态，如当前时刻 CPU 的使用率、内存的使用量、硬盘的容量以及GC次数等等。因为此类型的特点是随着时间的推移不断，值（相对而言）没有规则的变化。在 Kratos 框架中，针对 RPC 每次请求的延迟（latency）就是一个 Gauges，一段时间内的 Gauges 就组合成了一个 [RollingGauges](https://github.com/go-kratos/kratos/blob/master/pkg/stat/metric/rolling_gauge.go#L10)；此外，Gauge可增可减，与Counter不一样，在Prometheus上通过Gauge，**可以不用经过内置函数直观的反映数据的变化情况**
![image](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/metrics/prometheus/gauges-1.png)
下图表示堆可分配的空间大小：
![image](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/metrics/prometheus/gauges-3.png)


以时间为横坐标、数值为纵坐标，如下面的内存的使用图，就是典型的采集使用 Gauge 形式的 Metrics：

![image](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/metrics/prometheus/gauges-2.png)

#### Counters

Counter 就是计数器，从数据量 `0` 开始累计计算，只能增加，或者保持不变（增加 `0`），典型对应的场景是：持续增加的访问量采样数据。Counter 一般从 `0` 开始，一直不断的累加，但有可能保持不变（在图中以一条水平线表示）。通过Counter指标可以统计HTTP请求数量，请求错误数，接口调用次数等单调递增的数据，同时可结合`increase`和`rate`等函数统计变化速率

以时间为横坐标、数值为纵坐标，可以画出用 Counters 形式的 Metrics：

![image](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/metrics/prometheus/counter-1.png)

同样的，在 Kratos 中，每个连接上 RPC 请求的总次数和错误次数，就是一个 [Counters](https://github.com/go-kratos/kratos/blob/master/pkg/stat/metric/rolling_counter.go)。

<font color="#dd0000"> 一定要注意的是：不要使用计数器来监控可能减少的值 </font>。例如，不要使用计数器来处理当前正在运行的进程数，而应该用 Gauge。Counter 主要有两个方法：

```golang
// 将 counter 值加 1
Inc()
// 将指定值加到 counter 值上，如果指定值 < 0 会 panic.
Add(float64)
```

####  Histograms
Histograms 意为直方图，Histogram 会在一段时间范围内对数据进行采样（通常是请求持续时间或响应大小等），并将其计入可配置的存储桶（Bucket）中。可以观察到指标在各个不同的区间范围的分布情况，可以观察到请求耗时在各个桶的分布。如下图：

![histogram](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/metrics/prometheus/histograms-1.png)
![histogram](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/metrics/prometheus/histograms-2.png)

有一点要注意的是，Histogram是累计直方图，即每一个桶的是只有上区间，例如下图表示小于`0.1ms`（`le="0.1"`）的请求数量是`18173`个，小于`0.2ms`（`le="0.2"`）的请求是`18182`个，在`le="0.2"`这个桶中是包含了`le="0.1"`这个桶的数据，如果要拿到`0.1ms~0.2ms`的请求数量，可以通过两个桶相减

![histogram](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/metrics/prometheus/histograms-3.png)

此外，在直方图中，还可以通过`histogram_quantile`函数求出百分位数，比如`P50`/`P90`/`P99`等数据

####  Summary
Summary也是用来做统计分析的，和Histogram区别在于，Summary直接存储的就是百分位数，如下所示：可以直观的观察到样本的中位数，如`P90`和`P99`：
![histogram](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/metrics/prometheus/summary-1.png)

再次强调下，Summary的百分位数是客户端计算好直接让Prometheus抓取的，不需要Prometheus计算，直方图是通过内置函数`histogram_quantile`在Prometheus服务端计算出来的

#### Histograms 的应用意义

在大多数情况下人们都倾向于使用某些量化指标的平均值，例如 CPU 的平均使用率、页面的平均响应时间。这种方式的问题很明显，以系统 API 调用的平均响应时间为例：如果大多数 API 请求都维持在 100ms 的响应时间范围内，而个别请求的响应时间需要 `5s`，那么就会导致某些 WEB 页面的响应时间落到中位数的情况，而这种现象被称为长尾问题。为了区分是平均的慢还是长尾的慢，最简单的方式就是按照请求延迟的范围进行分组。例如，统计延迟在 `0~10ms` 之间的请求数、延迟在 `10~20ms` 之间的请求数等等。通过这种方式可以快速分析系统慢的原因。Histogram 和 Summary 都是为了能够解决这样问题的存在，通过 Histogram 和 Summary 类型的监控指标，可以快速了解监控样本的分布情况。

## 0x03  指标操作/开发

####  指标导出
1. 使用exporter
2. 利用[官方库](github.com/prometheus/client_golang/prometheus/promhttp)实现，笔者较常用

代码示例，见[仓库](https://github.com/pandaychen/golang_in_action/tree/master/prometheus)

## 0x04 PromQL介绍

## 0x05  Grafana可视化

## 0x06 参考

- [一文带你了解 Prometheus](https://cloud.tencent.com/developer/article/1999843)
- [一文搞懂 Prometheus 的直方图](https://juejin.im/post/5d492d1d5188251dff55b0b5)
- [如何区分 prometheus 中 Histogram 和 Summary 类型的 metrics？](https://www.cnblogs.com/aguncn/p/9920545.html)
- [Metrics 设计：teleport](https://goteleport.com/teleport/docs/metrics-logs-reference/)
- [Lock-free Observations for Prometheus Histograms](https://grafana.com/blog/2020/01/08/lock-free-observations-for-prometheus-histograms/)
- [golang API](https://godoc.org/github.com/prometheus/client_golang/prometheus)

转载请注明出处，本文采用 [CC4.0](http://creativecommons.org/licenses/by-nc-nd/4.0/) 协议授权
