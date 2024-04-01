---
layout: post
title: prometheus 查询 && 应用实践（进阶篇）
subtitle: 如何更好的使用 prometheus（With Promql）
date: 2022-05-01
header-img: img/super-mario.jpg
author: pandaychen
catalog: true
tags:
  - Prometheus
  - Metrics
---

##  0x00    前言
前文[理解 Prometheus 的基本数据类型及应用（基础篇）](https://pandaychen.github.io/2020/04/11/PROMETHEUS-METRICS-INTRO/)梳理了Prometheus的应用基础，本文梳理下 prometheus 查询及指标应用的最佳实践以及Promql使用等

##  0x01    相关概念回顾

####    时序数据库

####    采样样本
Prometheus 会定期去对数据进行采集，每一次采集的结果都是一次采样样本（sample），这些数据会被存储为时间序列，也就是带有时间戳的 value stream，这些 value stream 归属于自己的监控指标。采集样本包括了 `3` 部分：
-   监控指标（metric）
-   毫秒时间戳（timestamp）
-   样本值（value）

####    关于监控指标的解读
监控指标被表示为下面的格式：

```text
metric_name {label_name_1=label_value_1, label_name_2=label_value_2, ...}
```

解释：
-   `metric_name`：指明监控的内容
-   `label_value_xxx`：用于声明这个监控内容中不同维度的值

使用常见的二维坐标系（xxx 坐标系）举例，有 `X`，`Y` 两个轴，上面有两点 `A` 和 `B`，坐标分别为 `(1, 3)` 和 `(2, 1)`：

```text
Y
  ^
  │   . A (1, 3)
  │
  │     . B (2, 1)
  |
    v------------------> X
```


对应于 Prometheus，`metric_name` 就是xxx 坐标系，而`label_name_1` 即为 `X`，`label_name_2` 为 `Y`。需要注意的是 `A `和 `B` 两个点并不代表采样点，而是**监控指标**。

**可以想象在此坐标系中还存在一条虚拟的时间轴，分别从 `A/B` 两点从屏幕外垂直屏幕进去，在这两条虚拟的时间轴上，每一个点就是一个采样点，采样点上会带一个毫秒时间戳和一个值，这个值就是样本的值**

对于 Prometheus 而言，这里存在两个时间序列，分别为：

-   坐标系 `{"X"="1","Y"="3"}`
-   坐标系 `{"X"="2","Y"="1"}`

在 Prometheus 中，样本的值必须为 `float64` 类型；此外，建议标签值不要使用一个数量非常多的值，否则会造成时间序列数量的极度膨胀。标签的值应该越简单越好

##  0x02  指标计算

####  常用函数整理
一个常用技巧是遇到`counter`数据类型，在做任何操作之前，先套上一个`rate()`或`increase()`函数

1、`rate`函数是专门搭配`counter`数据类型使用函数，功能是取`counter`在这个时间段中平均每秒的增量，例如获取`eth0`网卡`1m`内每秒流量的平均值，指标名是`node_network_receive_bytes_total`，label选择`eth0`

```BASH
rate(node_network_receive_bytes_total{device="eth0"}[1m])
```

2、`increase`函数表示某段时间内数据的增量，而`rate()` 函数则表示某段时间内数据的平均值；那么这两个函数如何选取使用呢？当获取数据比较精细的时候，类似于`1m`取样推荐使用`rate()`函数；当获取数据比较粗糙的时候类似于`5m`/`10m`甚至更长时间取样推荐使用`increase()`函数；例如获取`eth0`网卡`1m`内流量的增量

```BASH
increase(node_network_receive_bytes_total{device="eth0"}[1m])
```

3、`sum`函数是求和函数,注意点是使用`sum`后是**将所有的监控的服务器的值进行取和**，所以只看某一台服务器指标时需要使用`by increase()`进行拆分。例如获取所有主机`eth0`网卡`1m`内每秒流量的平均值的和

```BASH
sum(rate(node_network_receive_bytes_total{device="eth0"}[1m]))
```

4、`topk`函数取前面`N`位的最高值（即topN），当有很多服务器我们想要获取某个key的数据排在前`3`位的服务器，可使用如下：

```BASH
topk(3,key) #FOR GUAGE
topk(3,rate(key[1m])) #FOR COUNTER
```

5、`count`函数是找出当前或者历史数据中某个key的数值大于或小于某个值的统计，例如：

```BASH
count(node_netstat_Tcp_CurrEstab >50)
```

6、`irate`函数


##  0x  实战：CPU使用率的计算
CPU模式，CPU要通过分时复用的方式运行于不同的模式中，使用`top`命令查看，通过`curl http://localhost:9100/metrics`拿到CPU的具体指标数据如下：

- `us`：用户进程使用cpu的时间
- `sy`：内核进程使用cpu的时间
- `ni`：用户进程空间内改变过优先级的进程使用的cpu时间
- `id`：空闲cpu时间
- `wa`：等待io的cpu时间
- `hi`：硬中断的cpu时间
- `si`：软中断的cpu时间
- `st`：虚拟机管理程序使用的cpu时间

某个时间点指标如下：

```TEXT
# HELP node_cpu_seconds_total Seconds the cpus spent in each mode.
# TYPE node_cpu_seconds_total counter
node_cpu_seconds_total{cpu="0",mode="idle"} 26659.41
node_cpu_seconds_total{cpu="0",mode="iowait"} 4.79
node_cpu_seconds_total{cpu="0",mode="irq"} 0
node_cpu_seconds_total{cpu="0",mode="nice"} 0
node_cpu_seconds_total{cpu="0",mode="softirq"} 2.69
node_cpu_seconds_total{cpu="0",mode="steal"} 0
node_cpu_seconds_total{cpu="0",mode="system"} 31.65
node_cpu_seconds_total{cpu="0",mode="user"} 8.67
node_cpu_seconds_total{cpu="1",mode="idle"} 26634.43
node_cpu_seconds_total{cpu="1",mode="iowait"} 54.14
node_cpu_seconds_total{cpu="1",mode="irq"} 0
node_cpu_seconds_total{cpu="1",mode="nice"} 0.02
node_cpu_seconds_total{cpu="1",mode="softirq"} 1.23
node_cpu_seconds_total{cpu="1",mode="steal"} 0
node_cpu_seconds_total{cpu="1",mode="system"} 34.07
node_cpu_seconds_total{cpu="1",mode="user"} 9
node_cpu_seconds_total{cpu="2",mode="idle"} 26629.89
node_cpu_seconds_total{cpu="2",mode="iowait"} 6.57
node_cpu_seconds_total{cpu="2",mode="irq"} 0
node_cpu_seconds_total{cpu="2",mode="nice"} 0
node_cpu_seconds_total{cpu="2",mode="softirq"} 1.95
node_cpu_seconds_total{cpu="2",mode="steal"} 0
node_cpu_seconds_total{cpu="2",mode="system"} 24.66
node_cpu_seconds_total{cpu="2",mode="user"} 7.2
node_cpu_seconds_total{cpu="3",mode="idle"} 26699.96
node_cpu_seconds_total{cpu="3",mode="iowait"} 5.72
node_cpu_seconds_total{cpu="3",mode="irq"} 0
node_cpu_seconds_total{cpu="3",mode="nice"} 0.01
node_cpu_seconds_total{cpu="3",mode="softirq"} 1.27
node_cpu_seconds_total{cpu="3",mode="steal"} 0
node_cpu_seconds_total{cpu="3",mode="system"} 22.32
node_cpu_seconds_total{cpu="3",mode="user"} 7.33
```

##  0x03  Review（2024）

####  四大指标的意义及常用应用

以http-cgi为例，定义如下指标
```go
var (
	httpRequestsTotal = promauto.NewCounterVec(
		prometheus.CounterOpts{
			Name: "http_requests_total",
			Help: "Number of HTTP requests",
		},
		[]string{"path", "method", "status"},
	)

	httpRequestDuration = promauto.NewHistogramVec(
		prometheus.HistogramOpts{
			Name: "http_request_duration_seconds",
			Help: "Duration of HTTP requests",
		},
		[]string{"path", "method", "status"},
	)

	httpInflightRequests = promauto.NewGaugeVec(
		prometheus.GaugeOpts{
			Name: "http_inflight_requests",
			Help: "Number of inflight HTTP requests",
		},
		[]string{"path", "method"},
	)

	httpResponseSizeBytes = promauto.NewSummaryVec(
		prometheus.SummaryOpts{
			Name: "http_response_size_bytes",
			Help: "Size of HTTP responses",
		},
		[]string{"path", "method", "status"},
	)
)

func handleRequest(w http.ResponseWriter, r *http.Request) {
	path := r.URL.Path
	method := r.Method
	status := "200" // 假设响应状态码为 200

	// Counter
	httpRequestsTotal.WithLabelValues(path, method, status).Inc()

	// Gauge
	httpInflightRequests.WithLabelValues(path, method).Inc()
	defer httpInflightRequests.WithLabelValues(path, method).Dec()

	// Histogram
	timer := prometheus.NewTimer(httpRequestDuration.WithLabelValues(path, method, status))
	defer timer.ObserveDuration()

	// Summary
	respSize := doWork()
	httpResponseSizeBytes.WithLabelValues(path, method, status).Observe(float64(respSize))

	w.WriteHeader(http.StatusOK)
}

func doWork() int {
	// 模拟处理请求
	time.Sleep(100 * time.Millisecond)
	return 512 // 假设响应大小为 512 字节
}
```


####   实际应用：基于alertmanager的高级配置
如何使用Conter实现基于增量的告警策略？借助于 PromQL编写适当的查询表达式，并将其配置为 Alertmanager 的告警规则来实现。下面使用 Prometheus Counter 监控 HTTP `5xx` 错误并在错误率超过阈值时触发告警，主要代码如下：

```GO
var httpRequestsTotal = promauto.NewCounterVec(prometheus.CounterOpts{
	Name: "http_requests_total",
	Help: "Total number of HTTP requests",
}, []string{"code", "method"})

func main() {
	http.Handle("/metrics", promhttp.Handler())
	http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		code := "200"
		if r.URL.Path == "/error" {
			code = "500"
		}
		httpRequestsTotal.WithLabelValues(code, r.Method).Inc()
		w.WriteHeader(200)
	})
	http.ListenAndServe(":8080", nil)
}
```

那么，使用如下PromQL来计算以过去 `5` 分钟内每分钟 HTTP `5xx` 错误的增量：

```TEXT
rate(http_requests_total{code=~"5.."}[5m])
```

最后，在 Prometheus 配置文件中创建一个告警规则，当过去 `5` 分钟内**每分钟的** HTTP `5xx` 错误率超过 `10` 时，触发告警：

```YAML
groups:
- name: example
  rules:
  - alert: HighErrorRate
    expr: rate(http_requests_total{code=~"5.."}[5m]) > 10
    for: 1m
    labels:
      severity: critical
    annotations:
      summary: "High error rate ({{ $value }} errors/min)"
      description: "HTTP 5xx error rate is too high, affecting the application"
```

####  label的意义
标签（label）是一种强大且灵活的元数据机制，用于对时间序列数据进行描述、区分和过滤。标签是键值对（key-value pair）形式的数据，可以附加到时间序列上，以提供有关数据来源和特性的详细信息。通过使用标签，Prometheus 可以更有效地查询和聚合数据，帮助开发人员和运维人员更好地理解和监控系统、应用程序和服务的性能

- 描述数据来源和特性：标签可以提供有关时间序列数据的详细描述，例如实例名称、服务名称、请求方法、状态码等
- 区分和过滤时间序列：标签允许你根据特定条件过滤和选择时间序列数据。例如，你可以使用标签查询特定服务或实例的性能指标，或者根据请求方法和状态码过滤 HTTP 请求数据
- 数据聚合：标签可以用于对数据进行分组和聚合，以便计算各种统计信息，如总和、平均值、最大值、最小值等

##  0x03 参考
-   [彻底理解 Prometheus 查询语法](https://blog.csdn.net/zhouwenjun0820/article/details/105823389)
-   [Prometheus 操作指南](https://github.com/yunlzheng/prometheus-book)
-   [理解时间序列](https://github.com/yunlzheng/prometheus-book/blob/master/promql/what-is-prometheus-metrics-and-labels.md)
-   [Prometheus Metrics 设计的最佳实践和应用实例，看这篇够了](https://cloud.tencent.com/developer/article/1639138)
- [PromQL 简明教程](https://www.kawabangga.com/posts/4408)
- [Prometheus技术分享——prometheus的函数与计算公式详解](https://zhuanlan.zhihu.com/p/595103670)
- [Prometheus的函数和计算公式](https://blog.csdn.net/wc1695040842/article/details/107013799)
- [监控metrics系列---- Prometheus入门](https://kingjcy.github.io/post/monitor/metrics/prometheus/prometheus/)
-   [PromQL 基本使用](https://songjiayang.gitbooks.io/prometheus/content/promql/summary.html)
-   [Prometheus 实战](https://songjiayang.gitbooks.io/prometheus/content/)