---
layout:     post
title:      Prometheus 应用接入：使用 Prometheus 开发 Exporter
subtitle:	如何在项目中使用 Prometheus 及 Exporter 开发基础
date:       2020-07-13
header-img: img/super-mario.jpg
author:     pandaychen
catalog:    true
tags:
    - Prometheus
---

##  0x00    前言
Prometheus 主要用于应用服务的监控，尤其是基于 Docker/Kubernetes 部署的应用服务，这里的监控是服务层面的（细粒度），以 Golang 开发的服务为例，如 `runtime` 信息，接口延迟，某个操作的延迟及接口调用成功率等等，只要是能够收集的信息，都可以作为 Prometheus 的监控指标。

##  0x01	应用接入
Prometheus 的应用接入其实并不复杂，通常按照如下几个步骤来完成：
1.	实现定好需要采集哪些指标，如成功率，延迟，分布数据等
2.	将指标转换为 Prometheus 的 Metrics 类型
3.	在代码逻辑中加入 Metrics 的 "打点" 调用
4.	启动 PrometheusHttp 服务，暴露自己的采集的指标

以上几步即可完成在项目应用中将指标暴露给 Prometheus

##	0x02	Exporter 简介
Exporter 是一个采集监控数据并通过 Prometheus 监控规范对外提供数据的组件，它负责从目标处（Your 服务）搜集数据，并将其转化为 Prometheus 支持的格式。Prometheus 会周期性地调用 Exporter 提供的 metrics 数据接口来获取数据。那么使用 Exporter 的好处是什么？

举例来说，如果要监控 Mysql/Redis 等数据库，我们必须要调用它们的接口来获取信息（前提要有），这样每家都有一套接口，这样非常不通用。所以 Prometheus 做法是每个软件做一个 Exporter，Prometheus 的 Http 读取 Exporter 的信息（将监控指标进行统一的格式化并暴露出来）。简单类比，Exporter 就是个翻译，把各种语言翻译成一种统一的语言。

![exporter](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/metrics/prometheus/exporter_principle.png)

Prometheus Exporter 就是用来收集和暴露指标的工具，通常情况下是 Prometheus Exporter 收集并暴露指标，然后 Prometheus 收集并存储指标，使用 Grafana 或者 Promethues UI 可以查询并展示指标。它包含两个重要的组件：

-	Collector：收集应用或者其他系统的指标，然后将其转化为 Prometheus 可识别收集的指标
-	Exporter：从 Collector 获取指标数据，并将其转成为 Prometheus 可读格式

Exporter 的功能是将数据周期性地从监控对象中取出来进行加工，然后将数据规范化后通过端点暴露给 Prometheus，主要流程图如下：
1.	封装功能模块获取监控系统内部的统计信息
2.	将返回数据进行规范化映射，使其成为符合 Prometheus 要求的格式化数据
3.	Collect 模块负责存储规范化后的数据，最后当 Prometheus 定时从 Exporter 提取数据时，Exporter 就将 Collector 收集的数据通过 HTTP 的形式在 `/metrics` 端点进行暴露

![flow](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/metrics/prometheus/exporter/exporter-flow.png)

##  0x03	Build Your Own Exportor：自定义 Collector
官方文档 [WRITING EXPORTERS](https://prometheus.io/docs/instrumenting/writing_exporters/) 介绍了编写 Exportor 的一些注意点。本文主要描述自定义 Collector 的实现。Prometheus 的 client 库提供了实现自定义 Exportor 的 [接口](https://github.com/prometheus/client_golang/blob/master/prometheus/collector.go#L27)，Collector 接口定义了两个方法 `Describe` 和 `Collect`，实现这两个方法就可以暴露自定义的数据。一般步骤如下：

1.	定义一个 Exporter 结构体，用于存放描述信息
2.	实现 `Collector` 接口
3.	实例化 Exporter
4.	注册指标
5.	暴露指标
6.	每被动采集时会调用 `Collector.Collect` 方法，获取到响应的指标数据

定义集群指标采集器---->数据采集工作---->实现`Describe`接口，写入描述信息---->将收集的数据导入到`Collector`---->结构体实例化赋值---->定义加注册指标---->注入自定义指标---->暴露metrics

collector开发例子[参考](https://github.com/pandaychen/golang_in_action/tree/master/prometheus/exporter)

####	Describe 接口
实现 `Describe` 接口，传递指标描述符到 channel
```golang
// Describe simply sends the two Descs in the struct to the channel.
func (c *ClusterManager) Describe(ch chan<- *prometheus.Desc) {
    ch <- c.OOMCountDesc
    ch <- c.RAMUsageDesc
}
```

####	Collect 接口
Collect 函数将执行抓取函数并返回数据，返回的数据传递到 channel 中，并且传递的同时绑定原先的指标描述符。实现指标采集的工作需要在这里完成（针对开发者）

```golang
type Collector interface {
	// Describe sends the super-set of all possible descriptors of metrics
	// collected by this Collector to the provided channel and returns once
	// the last descriptor has been sent. The sent descriptors fulfill the
	// consistency and uniqueness requirements described in the Desc
	// documentation.
	//
	// It is valid if one and the same Collector sends duplicate
	// descriptors. Those duplicates are simply ignored. However, two
	// different Collectors must not send duplicate descriptors.
	//
	// Sending no descriptor at all marks the Collector as “unchecked”,
	// i.e. no checks will be performed at registration time, and the
	// Collector may yield any Metric it sees fit in its Collect method.
	//
	// This method idempotently sends the same descriptors throughout the
	// lifetime of the Collector. It may be called concurrently and
	// therefore must be implemented in a concurrency safe way.
	//
	// If a Collector encounters an error while executing this method, it
	// must send an invalid descriptor (created with NewInvalidDesc) to
	// signal the error to the registry.
	Describe(chan<- *Desc)
	// Collect is called by the Prometheus registry when collecting
	// metrics. The implementation sends each collected metric via the
	// provided channel and returns once the last metric has been sent. The
	// descriptor of each sent metric is one of those returned by Describe
	// (unless the Collector is unchecked, see above). Returned metrics that
	// share the same descriptor must differ in their variable label
	// values.
	//
	// This method may be called concurrently and must therefore be
	// implemented in a concurrency safe way. Blocking occurs at the expense
	// of total performance of rendering all registered metrics. Ideally,
	// Collector implementations support concurrent readers.
	Collect(chan<- Metric)
}
```

####  暴露 Exportor 及 Promhttp 启动
使用下面的代码将自定义的 Metrics 暴露出去，这也是 promhttp 的通用做法了：
```golang
import (
    "log"
    "net/http"
    "github.com/prometheus/client_golang/prometheus/promhttp"
)

func main() {
    http.Handle("/metrics", promhttp.Handler())
    log.Fatal(http.ListenAndServe(":8080", nil))
}
```

##	0x04	Review：Collector 使用的场景
回顾下，有两种指标的接入方式：
1.	创建指标的时候声明了指标描述（exporter）
2.	先声明描述，再创建指标（collector）

```GO
// exporter
//1. 创建指标
HTTPReqTotal = prometheus.NewCounterVec(prometheus.CounterOpts{
        Name: "http_requests_total",
        Help: "Total number of HTTP requests",
    }, []string{"method", "path", "status"})
//2.	指标注册到 DefaultRegisterer
prometheus.MustRegister(
        HTTPReqTotal,
)
//3. 通过 HTTP API 暴露指标，当 promhttp.Handler() 被执行时，所有 metric 被序列化输出
http.Handle("/metrics", promhttp.Handler())
```

####	指标采集的触发时机
一般而言，对于 Counter/Histogram 这两种指标，触发时机比较明确，通常在某个方法结束前后运行即可，典型的场景如下：

```go
//QPS， 请求完成以后，请求数量直接累加
HTTPReqTotal.With(prometheus.Labels{
	PromLabelMethod: "GET",
	PromLabelPath:   "/api/query",
	PromLabelStatus: "200",
}).Inc()

// 监控请求耗时分布，请求完成后直接 observe
HTTPReqDuration.With(prometheus.Labels{
            PromLabelMethod: "GET",
            PromLabelPath:   "/api/query",
}).Observe(3.15)
```

那么什么情况下，使用第 2 种方式来进行指标处理呢？设想需要使用 Gauge 指标来监控某个 channel 的长度，有哪些可行的方式？

1.	启动定时器，定时器触发时调用 `guageMetric.Set`
2.	使用 `NewGaugeFunc` 实现
3.	自定义 `Collector` 实现

简单描述下这三种方式的实现代码：

```GO
//1: 定时任务
var jobChannel chan int
jobChanLength:= prometheus.NewGauge(prometheus.GaugeOpts{
        Name: "job_channel_length",
        Help: "length of job channel",
})
go func() {
	jobChanLength.Set(float64(len(jobChannel)))
	c := time.Tick(30 * time.Second)
	for range c {
		jobChanLength.Set(float64(len(jobChannel)))
	}
}()

//2：只需要定义函数 func()float64{} 即可，当 promhttp.Handler() 被调用时，func()float64{} 被触发
jobChanLength := prometheus.NewGaugeFunc(prometheus.GaugeOpts{
        Name: "job_channel_length",
        Help: "length of job channel",
},func() float64{
	return float64(len(jobChannel))
})
prometheus.MustRegister(jobChanLength)

//3: 自定义 collector

//Collect is called by the Prometheus registry when collecting metrics
type Collector interface {
    Describe(chan<- *Desc)
    // 比如满足并发安全
    Collect(chan<- Metric)
}

type ChanCollector struct{
    JobChannel chan int
    Name string
}

func NewChanCollector(jobChannel chan int, name string) *ChanCollector{
    c := ChanCollector{}
    c.JobChannel = jobChannel
    c.Name = name
    return &c
}

func (c *ChanCollector) Describe(ch chan<- *prometheus.Desc) {
    desc := prometheus.NewDesc(c.Name, "length of channel", nil, nil)
    ch <- desc
}

func (c *ChanCollector) Collect(ch chan<- prometheus.Metric) {
    ch <- prometheus.MustNewConstMetric(
        prometheus.NewDesc(c.Name, "length of channel", nil, nil),
        prometheus.GaugeValue,
        float64(len(c.JobChannel)),
    )
}

collector := NewChanCollector(jobChannel, "job_channel_length")
prometheus.MustRegister(collector)
```

注意，当使用自定义 `Collector` 实现指标收集，当 `promhttp.Handler()` 被调用时，已经注册 `MustRegister` 指标的 `Collect` 方法被执行，所以此函数不能运行时间过长，否则可能导致 `promhttp.Handler()` 超时

直接使用 Collector 收集指标时，go client Colletor 只会在每次响应 Prometheus 请求的时候才收集数据。需要每次显式传递变量的值，否则就不会再维持该变量，在 Prometheus 也将看不到这个变量。Collector 是一个 `interface`，所有收集 metrics 数据的对象都需要实现该接口。它内部提供了两个函数，`Collector` 用于收集用户数据，将收集好的数据传递给传入参数 `channel` 就可；`Descirbe` 函数用于描述这个 `Collector`；当收集系统收集的数据太多时时，就可以自定义 Collector 收集的方式来优化流程，并且在某些情况下如果已经有了一个可用的 metrics，就不需要使用 Counter/Gauage 等这些数据结构，直接在 Collector 内部实现一个代理的功能即可（相当于 metrics 转发功能）

##  0x04	一个完整的例子
本小节给出一个完整的 Exportor 例子，结构和接口的使用方法可以参考 [GODOC](https://godoc.org/github.com/prometheus/client_golang/prometheus)。
需要引入的 package：
```golang
import (
   "net/http"
   "github.com/prometheus/client_golang/prometheus/promhttp"
   "github.com/prometheus/common/log"
   "github.com/prometheus/client_golang/prometheus"
)
```

封装了 `fooCollector`，采集两个指标 `fooMetric` 和 `barMetric`：
```golang
type fooCollector struct {
   fooMetric *prometheus.Desc
   barMetric *prometheus.Desc
}
```

初始化 `newFooCollector`，注意 `prometheus.NewDesc` 的参数，推荐还是传入 `help` 及 `label` 信息：
```golang
func newFooCollector() *fooCollector {
   m1 := make(map[string]string)
   m1["env"] = "prod"
   v := []string{"hostname"}
   return &fooCollector{
      fooMetric: prometheus.NewDesc("fff_metrics","Show metrics a for mysql",nil,nil),
      barMetric:prometheus.NewDesc("bbb_metrics","Show metrics a bar occu",v,m1),
   }
}
```

实现 `Describe` 及 `Collect` 方法：
```golang
func (collect *fooCollector) Describe(ch chan <- *prometheus.Desc) {
   ch <- collect.barMetric
   ch <- collect.fooMetric

}

func (collect *fooCollector) Collect(ch chan<- prometheus.Metric) {
   var metricValue float64

   if 1 == 1 {
      metricValue = 1
   }


   ch <- prometheus.MustNewConstMetric(collect.fooMetric,prometheus.GaugeValue, metricValue)
   ch <- prometheus.MustNewConstMetric(collect.barMetric,prometheus.CounterValue, metricValue,"kk")
}
```

最后一个步骤，注册 `fooCollector` 及启动 `promhttp` 服务，以上几步就实现了一个简单的 `Exportor`
```golang
func main() {
   foo := newFooCollector()
   prometheus.MustRegister(foo)
   log.Info("beging to server on Port: 18080")
   http.Handle("/metrics",promhttp.Handler())
   log.Fatal(http.ListenAndServe(":18080",nil))
}
```

##  0x05	抽象 Exportor 的实现步骤
1、定义指标及初始化方法 <br>

定义指标就是创建指标的描述符，通常把要采集的指标描述符放在一个结构体里，如下我们定义了一个指标 `fooCollector`，其中包含了两种类型的数据 `fooMetric` 和 `barMetric`：

```golang
//Define a struct for you collector that contains pointers
//to prometheus descriptors for each metric you wish to expose.
//Note you can also include fields of other types if they provide utility
//but we just won't be exposing them as metrics.
type fooCollector struct {
	fooMetric *prometheus.Desc
	barMetric *prometheus.Desc
}
```

初始化方法 `newFooCollector` 用于初始化 `fooCollector`，包含对每个成员 `prometheus.Desc` 类型的初始化，`prometheus.NewDesc` 方法创建一个 `prometheus.Desc` 对象：
```golang
//You must create a constructor for you collector that
//initializes every descriptor and returns a pointer to the collector
// 调用工厂方法即可创建一个结构体的实例
func newFooCollector() *fooCollector {
	return &fooCollector{
		fooMetric: prometheus.NewDesc("foo_metric",
			"Shows whether a foo has occurred in our cluster",
			nil, nil,
		),
		barMetric: prometheus.NewDesc("bar_metric",
			"Shows whether a bar has occurred in our cluster",
			nil, nil,
		),
	}
}
```

2、实现数据采集（核心）
数据采集需要实现 Collector 的两个接口：
-   `Describe`：告诉 `prometheus` 我们定义了哪些 `prometheus.Desc` 结构，通过 `channel` 传递给上层
-   `Collect`：真正实现数据采集的功能，将采集数据结果通过 channel 传递给上层

注意：`Collect` 中完成的就是每个指标的采集，必须保证这里的数据是动态更新的，每一次调用 `curl http://127.0.0.1:8080/metrics` 都会触发对 `Collect` 方法的调用

```golang
//Each and every collector must implement the Describe function.
//It essentially writes all descriptors to the prometheus desc channel.
func (collector *fooCollector) Describe(ch chan<- *prometheus.Desc) {
	//Update this section with the each metric you create for a given collector
	ch <- collector.fooMetric
	ch <- collector.barMetric
}

//Collect implements required collect function for all promehteus collectors
func (collector *fooCollector) Collect(ch chan<- prometheus.Metric) {
	//Implement logic here to determine proper metric value to return to prometheus
	//for each descriptor or call other functions that do so.
	var metricValue float64
	if 1 == 1 {
		metricValue = 1
	}

	fmt.Println("start Collect")

	//Write latest value for each metric in the prometheus metric channel.
	//Note that you can pass CounterValue, GaugeValue, or UntypedValue types here.
	ch <- prometheus.MustNewConstMetric(collector.fooMetric, prometheus.CounterValue, metricValue)
	ch <- prometheus.MustNewConstMetric(collector.barMetric, prometheus.CounterValue, metricValue)
}
```

注意：每次调用 `curl http://127.0.0.1:8080/metrics` 时，都会触发一次已注册 `MustRegister` 到 Prometheus 指标的 `Collect` 方法的调用，注册了多少个指标，都会调用其 `Collect` 方法

3、注册指标及启动 `promHTTP` 服务
这里在主函数中注册上面自定义的指标，注册成功之后，启动 HTTP 服务器，这样就完成了自定义的 Exportor 服务。
```golang
func main() {
    //Create a new instance of the foocollector and
    //register it with the prometheus client.
    foo := newFooCollector()
    prometheus.MustRegister(foo)

    //This section will start the HTTP server and expose
    //any metrics on the /metrics endpoint.
    http.Handle("/metrics", promhttp.Handler())
    log.Info("Beginning to serve on port :8080")
    log.Fatal(http.ListenAndServe(":8080", nil))
}
```

4、最后，通过 `curl http://127.0.0.1:8080/metrics` 即可查看暴露的指标

```bash
# HELP bar_metric Shows whether a bar has occurred in our cluster
# TYPE bar_metric gauge
bar_metric 0.07074170776466579 1720775972352
# HELP foo_metric Shows whether a foo has occurred in our cluster
# TYPE foo_metric gauge
foo_metric 0.07074170776466579 1720772372352
```

##	0x06	一个实践例子：sarama
使用 [sarama](https://github.com/IBM/sarama) 库，由于其内置的指标实现是基于 `github.com/rcrowley/go-metrics` 库的，现在想把其转换为 Prometheus 的格式，如何实现？

参考 [saramaprom](https://github.com/iimos/saramaprom) 的做法，这里简单分析下其实现。

1、调用方法如下，把 sarama 的 `cfg.MetricRegistry` 作为参数传入 `saramaprom.ExportMetrics` 方法

```GO
ctx := context.Background()
cfg := sarama.NewConfig()
err := saramaprom.ExportMetrics(ctx, cfg.MetricRegistry, saramaprom.Options{
	Label: "some name to distinguish between different sarama instances",
})
```

2、`customCollector` 实现

```GO
// for collecting prometheus.constHistogram objects
type customCollector struct {
	prometheus.Collector

	metric prometheus.Metric
	mutex  *sync.Mutex
}

func newCustomCollector(mu *sync.Mutex) *customCollector {
	return &customCollector{
		mutex: mu,
	}
}

func (c *customCollector) Collect(ch chan<- prometheus.Metric) {
	c.mutex.Lock()
	if c.metric != nil {
		val := c.metric
		ch <- val
	}
	c.mutex.Unlock()
}

func (c *customCollector) Describe(_ chan<- *prometheus.Desc) {
	// empty method to fulfill prometheus.Collector interface
}
```

3、`exporter` 结构

```GOLANG
type exporter struct {
	opt              Options
	registry         MetricsRegistry
	promRegistry     prometheus.Registerer
	gauges           map[string]prometheus.Gauge
	customMetrics    map[string]*customCollector	// 可能有多个 customCollector
	histogramBuckets []float64
	timerBuckets     []float64		//time bucket
	mutex            *sync.Mutex
}
```

4、初始化 `exporter` 时，会异步启动收集 && 转换逻辑，这也是本项目的核心逻辑，主要目的是定时把 `go-metrics` 的格式转换为 Prometheus 的格式

-	通过 `c.registry.Each` 方法，定时从 sarama 库暴露的指标收集所有的数据
-	按照 metrics 的类型，将其转换为 Prometheus 的标准格式，`gaugeFromNameAndValue` 和 `gaugeFromNameAndValue`

```golang
func (c *exporter) update() error {
	var err error
	c.registry.Each(func(name string, i interface{}) {
		switch metric := i.(type) {
		case metrics.Counter:
			err = c.gaugeFromNameAndValue(name, float64(metric.Count()))
		case metrics.Gauge:
			err = c.gaugeFromNameAndValue(name, float64(metric.Value()))
		case metrics.GaugeFloat64:
			err = c.gaugeFromNameAndValue(name, float64(metric.Value()))
		case metrics.Histogram: // sarama
			samples := metric.Snapshot().Sample().Values()
			if len(samples) > 0 {
				lastSample := samples[len(samples)-1]
				err = c.gaugeFromNameAndValue(name, float64(lastSample))
			}
			if err == nil {
				err = c.histogramFromNameAndMetric(name, metric, c.histogramBuckets)
			}
		case metrics.Meter: // sarama
			lastSample := metric.Snapshot().Rate1()
			err = c.gaugeFromNameAndValue(name, float64(lastSample))
		case metrics.Timer:
			lastSample := metric.Snapshot().Rate1()
			err = c.gaugeFromNameAndValue(name, float64(lastSample))
			if err == nil {
				err = c.histogramFromNameAndMetric(name, metric, c.timerBuckets)
			}
		}
	})
	return err
}
```

5、`gaugeFromNameAndValue` 的实现，这个是直接通过指标暴露

```GO
func (c *exporter) gaugeFromNameAndValue(name string, val float64) error {
	shortName, labels, skip := c.metricNameAndLabels(name)
	if skip {
		if c.opt.Debug {
			fmt.Printf("[saramaprom] skip metric %q because there is no broker or topic labels\n", name)
		}
		return nil
	}

	if _, exists := c.gauges[name]; !exists {
		// 不存在则新建
		labelNames := make([]string, 0, len(labels))
		for labelName := range labels {
			labelNames = append(labelNames, labelName)
		}

		g := prometheus.NewGaugeVec(prometheus.GaugeOpts{
			Namespace: c.sanitizeName(c.opt.Namespace),
			Subsystem: c.sanitizeName(c.opt.Subsystem),
			Name:      c.sanitizeName(shortName),
			Help:      shortName,
		}, labelNames)

		if err := c.promRegistry.Register(g); err != nil {
			switch err := err.(type) {
			case prometheus.AlreadyRegisteredError:
				var ok bool
				g, ok = err.ExistingCollector.(*prometheus.GaugeVec)
				if !ok {
					return fmt.Errorf("prometheus collector already registered but it's not *prometheus.GaugeVec: %v", g)
				}
			default:
				return err
			}
		}
		c.gauges[name] = g.With(labels)
	}

	// 存在就直接 set
	c.gauges[name].Set(val)
	return nil
}
```

6、`histogramFromNameAndMetric` 的实现，与 `gaugeFromNameAndValue` 不同的是，该方法是通过 exporter 的方式暴露的（原因）

```GO
func (c *exporter) histogramFromNameAndMetric(name string, goMetric interface{}, buckets []float64) error {
	key := c.createKey(name)
	collector, exists := c.customMetrics[key]
	if !exists {
		collector = newCustomCollector(c.mutex)
		c.promRegistry.MustRegister(collector)
		c.customMetrics[key] = collector
	}

	var ps []float64
	var count uint64
	var sum float64
	var typeName string

	switch metric := goMetric.(type) {
	case metrics.Histogram:
		snapshot := metric.Snapshot()
		ps = snapshot.Percentiles(buckets)
		count = uint64(snapshot.Count())
		sum = float64(snapshot.Sum())
		typeName = "histogram"
	case metrics.Timer:
		snapshot := metric.Snapshot()
		ps = snapshot.Percentiles(buckets)
		count = uint64(snapshot.Count())
		sum = float64(snapshot.Sum())
		typeName = "timer"
	default:
		return fmt.Errorf("unexpected metric type %T", goMetric)
	}

	bucketVals := make(map[float64]uint64)
	for ii, bucket := range buckets {
		bucketVals[bucket] = uint64(ps[ii])
	}

	name, labels, skip := c.metricNameAndLabels(name)
	if skip {
		return nil
	}

	desc := prometheus.NewDesc(
		prometheus.BuildFQName(
			c.sanitizeName(c.opt.Namespace),
			c.sanitizeName(c.opt.Subsystem),
			c.sanitizeName(name)+"_"+typeName,
		),
		c.sanitizeName(name),
		nil,
		labels,
	)

	hist, err := prometheus.NewConstHistogram(desc, count, sum, bucketVals)
	if err != nil {
		return err
	}
	c.mutex.Lock()
	collector.metric = hist	// 存储在一个 map 中，value 为 collector 类型
	c.mutex.Unlock()
	return nil
}
```


其他exporter实现：
-	[Exporter for MySQL server metrics](https://github.com/prometheus/mysqld_exporter)
-	[Prometheus Exporter Knowledge Hub](https://exporterhub.io/)
-	[ssh_exporter](https://github.com/treydock/ssh_exporter/tree/master)
-	[apache_exporter](https://github.com/Lusitaniae/apache_exporter/blob/master/collector/collector.go)：该项目的collector实现非常典型，值得阅读

##	0x07	总结
本篇文章分析了 Prometheus 在应用中的接入方法及实现的步骤。此外，在自定义collector的实现中，对于`prometheus.Desc`/`prometheus.Counter`/`prometheus.GaugeVec`这几种典型的指标上报，要特别注意，对于`prometheus.GaugeVec`这种自定义标签类型，通常的实现方式参考（典型的代码）：

```go
type Exporter struct {
	  host       string

	  version   *prometheus.Desc
	  total     *prometheus.GaugeVec
	  available *prometheus.GaugeVec
	  logger    log.Logger
  }

func NewExporter(logger log.Logger, config *Config) *Exporter {
	return &Exporter{
		host:       config.host,
		logger:     logger,
		version: prometheus.NewDesc(
			prometheus.BuildFQName(namespace, "", "version"),
			"demo exporter version",
			nil,
			nil),
		total: prometheus.NewGaugeVec(
			prometheus.GaugeOpts{
				Namespace: namespace,
				Name:      "total",
				Help:      "demo total",
			},
			[]string{"region_id", "instance_id"},
		), // 定义gauge类型带标签region_id, instance_id的数据结构
	}
}

func (e *Exporter) Describe(ch chan<- *prometheus.Desc) {
	  ch <- e.version
	  e.total.Describe(ch)
  }

func (e *Exporter) Collect(ch chan<- prometheus.Metric) {
	ch <- prometheus.MustNewConstMetric(e.version, prometheus.GaugeValue, 0.8)
	e.total.Reset()

	start++
	e.total.With(prometheus.Labels{"region_id": "region1", "instance_id": "instance1"}).Set(float64(start))
	e.total.Collect(ch)
}
```

对于有动态标签的gauge的，需要通过`With(xxx).Set()`方法设置指标的值，并且需要使用`e.total.Collect(ch)` 方式去更新，不需要再把val传递到channel了

##  0x08	参考
-   [WRITING EXPORTERS](https://prometheus.io/docs/instrumenting/writing_exporters/)
-   [INSTRUMENTING A GO APPLICATION FOR PROMETHEUS](https://prometheus.io/docs/guides/go-application/)
-   [EXPORTERS AND INTEGRATIONS](https://prometheus.io/docs/instrumenting/exporters/)
-   [Go 语言开发 Prometheus Exporter 示例](https://blog.csdn.net/lisong694767315/article/details/81743555)
-   [Building a custom Prometheus exporter](https://blog.skyrise.tech/custom-prometheus-exporter)
-	[玩转 PROMETHEUS(6) 实现自定义的 COLLECTOR](https://vearne.cc/archives/39380)