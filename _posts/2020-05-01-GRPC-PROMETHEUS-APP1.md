---
layout:     post
title:      gRPC 应用之 Prometheus 监控接入
subtitle:   使用 gRPC 拦截器对接 Prometheus 监控介绍
date:       2020-05-01
author:     pandaychen
header-img:
catalog: true
category:   false
tags:
    - Prometheus
---

##  0x00    前言
&emsp;&emsp; 本文的出发点是希望在 gRPC 的框架中，加入核心数据监控的能力（Monitoring），目前预研的库有两个：
-   prometheus
-   open-falcon

这篇文章，介绍下如何将 gRPC 和 Prometheus 结合。Prometheus 的架构中常用的是 Pull 模式，当然 Push 也可以。这其实就是服务端（服务注册）和客户端（主动上报数据）的区别。
在 Pull 模式中, 通过 HTTP 协议去采集指标，只要应用系统能够提供 HTTP 接口（一般使用 promhttp 创建 HttpServer）就可以接入监控系统，相比于私有协议或二进制协议来说更为简单。


##  0x01    Prometheus Metrics

####    基础Metrics类型
&emsp;&emsp; Prometheus 常用 4种 Metric 类型，前四种为：Counter，Gauge，Summary 和 Histogram，每种类型都有对应的 Vector 版本：GaugeVec, CounterVec, SummaryVec, HistogramVec，vector 版本细化了 prometheus 数据模型，增加了 label 维度。
只有基础 metric 类型实现了 Metric 接口，metric 和它们的 vector 版本都实现了 collector 接口。collector 负责一系列 metrics 的采集，但是为了方便，metric 也可以 “收集自己”。注意：Gauge, Counter, Summary, Histogram, 和 Untyped 自身就是接口，而 GaugeVec, CounterVec, SummaryVec, HistogramVec, 和 UntypedVec 则不是接口。
为了创建 metric 和它们的 vector 版本，需要选择合适的 opts 结构体，如 GaugeOpts, CounterOpts, SummaryOpts, HistogramOpts, 或 UntypedOpts.

####    Custom Collectors and constant Metrics
实现自己的 metric，一般只需要实现自己的 collector 即可。如果已经有了现成的 metric（prometheus 上下文之外创建的），则无需使用 Metric 类型接口，只需要在采集期间将现有的 metric 映射到 prometheus metric 即可，此时可以使用 NewConstMetric, NewConstHistogram, and NewConstSummary (以及对应的 Must… 版本) 来创建 metric 实例，以上操作在 collect 方法中实现。describe 方法用于返回独立的 Desc 实例，NewDesc 用于创建这些 metric 实例。（NewDesc 用于创建 prometheus 识别的 metric）
如果只需要调用一个函数来收集一个 float 值作为 metric，那么推荐使用 GaugeFunc, CounterFunc, 或 UntypedFunc

####    Advanced Uses of the Registry
MustRegister 是注册 collector 最通用的方式。如果需要捕获注册时产生的错误，可以使用 Register 函数，该函数会返回错误。
如果注册的 collector 与已经注册的 metric 不兼容或不一致时就会返回错误。registry 用于使收集的 metric 与 prometheus 数据模型保持一致。不一致的错误会在注册时而非采集时检测到。前者会在系统的启动时检测到，而后者只会在采集时发生（可能不会在首次采集时发生），这也是为什么 collector 和 metric 必须向 Registry describe 它们的原因。
以上提到的 registry 都被称为默认 registry，可以在全局变量 DefaultRegisterer 中找到。使用 NewRegistry 可以创建 custom registry，或者可以自己实现 Registerer 或 Gatherer 接口。custom registry 的 Register 和 Unregister 运作方式类似，默认 registry 则使用全局函数 Register 和 Unregister。
custom registry 的使用方式还有很多：可以使用 NewPedanticRegistry 来注册特殊的属性；可以避免由 DefaultRegisterer 限制的全局状态属性；也可以同时使用多个 registry 来暴露不同的 metrics。
DefaultRegisterer 注册了 Go runtime metrics （通过 NewGoCollector）和用于 process metrics 的 collector（通过 NewProcessCollector）。通过 custom registry 可以自己决定注册的 collector

#####   HTTP Exposition
Registry 实现了 Gather 接口。调用 Gather 接口可以通过某种方式暴露采集的 metric。通常 metric endpoint 使用 http 来暴露 metric。通过 http 暴露 metric 的工具为 promhttp 子包。


##  0x02    Prometheus 包的用法
&emsp;&emsp; Prometheus 包提供了用于实现监控代码的 Metric 原型和用于注册 Metric 的 Registry。Promhttp 允许通过 HTTP 来暴露注册的 Metric 或将注册的 Metric 推送到 Pushgateway。

在 Prometheus 中，一般 Metric 对象采集的使用步骤如下：
1.    初始化一个 Metric 对象
2.    Register 注册对象
3.    向对象中添加值


####    常用类型及方法
```golang
type AlreadyRegisteredError         // 该类型实现了 error 接口，由 Register 返回，用于判断用于注册的 collector 是否已经被注册过
type Collector                      // 用于采集 prometheus metric，如果运行多个相同的实例，则需要使用 ConstLabels 来注册这些实例。实现 collector 接口需要实现 Describe 和 Collect 方法，并注册 collector
type Registerer                     // 负责 collector 的注册和去注册，实现 custom registrer 时应该实现该接口

func Register(c Collector) error    // 使用 DefaultRegisterer 来注册传入的 Collector
func Unregister(c Collector) bool   // 使用 DefaultRegisterer 来移除传入的 Collector 的注册信息
```

此外，带 Must 的版本函数只是对不带 Must 函数的封装，增加了 panic 操作，如：
```golang
// MustRegister implements Registerer.
func (r *Registry) MustRegister(cs ...Collector) {
　　for _, c := range cs {
　　　　if err := r.Register(c); err != nil {
　　　　　　panic(err)
　　　　}
　　}
}
```

##  0x03    Counter && CouterVec
计数器（Counter）是表示单个单调递增计数器的累积量，其值只能增加或在重启时重置为零。 例如，可以使用计数器来表示服务的总请求数，已完成的任务或错误总数。 但是注意不要使用计数器来监控可能减少的值。
Counter 主要两个方法，`Inc` 和 `Add`,
```golang
Inc()   // 将 counter 值加 1
Add(float64)    // 将指定值加到 counter 值上，如果指定值 < 0 会 panic
```

####    Counter使用例子
```golang
//初始化一个Counter
pushCounter = prometheus.NewCounter(prometheus.CounterOpts{
    Name: "repository_pushes",
    Help: "Number of pushes to external repository.",   //注意，必须要定义Help，否则会报错
})

//注册
err = prometheus.Register(pushCounter)
if err != nil {
    fmt.Println("Push counter couldn't be registered AGAIN, no counting will happen:", err)
    return
}

collectChan := make(chan struct{})

for range collectChan {
    //向对象中写入（统计）值
    pushCounter.Inc()
}
```

####    CounterVec使用例子



##  0x01    参考
-   [Go gRPC Interceptors for Prometheus monitoring](https://github.com/grpc-ecosystem/go-grpc-prometheus)
-   [](https://segmentfault.com/a/1190000018372347)
-   [](https://godoc.org/github.com/prometheus/client_golang/prometheus)