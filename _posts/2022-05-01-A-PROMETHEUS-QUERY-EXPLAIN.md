---
layout: post
title: prometheus 查询 && 应用实践（进阶篇）
subtitle: 如何更好的使用 prometheus（With PromQL）
date: 2022-05-01
header-img: img/super-mario.jpg
author: pandaychen
catalog: true
tags:
  - Prometheus
  - Metrics
---

##  0x00    前言
前文 [理解 Prometheus 的基本数据类型及应用（基础篇）](https://pandaychen.github.io/2020/04/11/PROMETHEUS-METRICS-INTRO/) 梳理了 Prometheus 的应用基础，本文梳理下 prometheus 查询及指标应用的最佳实践以及 PromQL 使用等

##  0x01    相关概念回顾

####    时序数据库

####    采样样本
Prometheus 会定期去对数据进行采集，每一次采集的结果都是一次采样样本（sample），这些数据会被存储为时间序列，也就是带有时间戳的 value stream，这些 value stream 归属于自己的监控指标。采集样本包括了 `3` 部分：
-   监控指标（metric）
-   毫秒时间戳（timestamp）
-   样本值（value）

####    关于监控指标的解读
监控指标被表示为下面的格式：

```python
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


对应于 Prometheus，`metric_name` 就是 xxx 坐标系，而 `label_name_1` 即为 `X`，`label_name_2` 为 `Y`。需要注意的是 `A ` 和 `B` 两个点并不代表采样点，而是 ** 监控指标 **。

** 可以想象在此坐标系中还存在一条虚拟的时间轴，分别从 `A/B` 两点从屏幕外垂直屏幕进去，在这两条虚拟的时间轴上，每一个点就是一个采样点，采样点上会带一个毫秒时间戳和一个值，这个值就是样本的值 **

对于 Prometheus 而言，这里存在两个时间序列，分别为：

-   坐标系 `{"X"="1","Y"="3"}`
-   坐标系 `{"X"="2","Y"="1"}`

在 Prometheus 中，样本的值必须为 `float64` 类型；此外，建议标签值不要使用一个数量非常多的值，否则会造成时间序列数量的极度膨胀。标签的值应该越简单越好

##  0x02  指标计算

####  常用函数整理
** 一个常用技巧是遇到 `counter` 数据类型，在做任何操作之前，先套上一个 `rate` 或 `increase` 函数 **

1、`rate` 函数是专门搭配 `counter` 数据类型使用函数，功能是取 `counter` 在这个时间段中平均每秒的增量，例如获取 `eth0` 网卡 `1m` 内每秒流量的平均值，指标名是 `node_network_receive_bytes_total`，label 选择 `eth0`

```python
rate(node_network_receive_bytes_total{device="eth0"}[1m])
```

2、`increase` 函数表示某段时间内数据的增量，而 `rate()` 函数则表示某段时间内数据的平均值；那么这两个函数如何选取使用呢？当获取数据比较精细的时候，类似于 `1m` 取样推荐使用 `rate()` 函数；当获取数据比较粗糙的时候类似于 `5m`/`10m` 甚至更长时间取样推荐使用 `increase()` 函数；例如获取 `eth0` 网卡 `1m` 内流量的增量
`increase` 增量是不做除法的，而 `rate` 要先计算增量后要除以当前的 offset 的偏移

```python
increase(node_network_receive_bytes_total{device="eth0"}[1m])
```

3、`sum` 函数是求和函数, 注意点是使用 `sum` 是 ** 将所有的监控的服务器的值进行取和 **，所以只看某一台服务器指标时需要使用 `by` 进行拆分。例如获取所有主机 `eth0` 网卡 `1m` 内每秒流量的平均值的和：

```python
sum(rate(node_network_receive_bytes_total{device="eth0"}[1m]))
```

4、`topk` 函数取前面 `N` 位的最高值（即 topN），当存在很多服务器时，想要获取某个 key 的数据排在前 `3` 位的服务器，可使用如下：

```python
topk(3,key) #FOR GUAGE
topk(3,rate(key[1m])) #FOR COUNTER
```

5、`count` 函数是找出当前或者历史数据中某个 key 的数值大于或小于某个值的统计，例如：

```python
count(node_netstat_Tcp_CurrEstab>50)
```

##  0x03  PromQL 实战：CPU 使用率的计算
本小节，以 CPU 使用率场景介绍下 PromQL 的使用，CPU 模式，CPU 要通过分时复用的方式运行于不同的模式中，使用 `top` 命令查看，通过 `curl http://localhost:9100/metrics` 拿到 CPU 的具体指标数据如下（通过 `node-exporter` 抓取的指标中 cpu 相关主要是各个 `node_cpu_seconds_total`）

- `us`：用户进程使用 cpu 的时间
- `sy`：内核进程使用 cpu 的时间
- `ni`：用户进程空间内改变过优先级的进程使用的 cpu 时间
- `id`：空闲 cpu 时间
- `wa`：等待 io 的 cpu 时间
- `hi`：硬中断的 cpu 时间
- `si`：软中断的 cpu 时间
- `st`：虚拟机管理程序使用的 cpu 时间

某个时间点指标如下：

```PYTHON
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

上面的一行就是某一核 cpu 的某个模式的运行时间（单位：s），把某一核各个模式的 cpu 时间加起来就是执行 `uptime` 得到的系统开机以来运行运行的总的秒数，比如 `node_cpu_seconds_total{cpu="0",mode="idle"} 26659.41` 意义是从系统开机到现在为止，`cpu0` 的空闲时间是 `26659.41s`，用它除以 `uptime` 就可得到开机以来 `cpu0` 的空闲率

基于上述指标，扩展计算 CPU 使用率的公式：

1、`cpu0` 在 `5` 分钟内处于空闲状态的时间：`increase(node_cpu_seconds_total{cpu="0",mode="idle"}[5m])`，该公式表示的是当前时间点的 `node_cpu_seconds_total` 减去 `5min` 之前的 `node_cpu_seconds_total` 的值，也就是这 `5min` 内处于 `idle` 状态的 cpu 时间

2、`cpu0` `5` 分钟内处于空闲状态的时间占比：`increase(node_cpu_seconds_total{cpu="0",mode="idle"}[5m]) / increase(node_cpu_seconds_total{cpu="0"}[5m])`

3、一台主机所有 `cpu` `5` 分钟内处于空闲状态的时间占比：`sum (increase(node_cpu_seconds_total{mode="idle"}[5m])) / sum (increase(node_cpu_seconds_total{mode="idle"}[5m]))`

4、若 Prometheus 监控多台主机（`instance`），要根据每台主机做 sum：`sum (increase(node_cpu_seconds_total{mode="idle"}[5m]))  by (instance) / sum (increase(node_cpu_seconds_total[5m])) by (instance)`

5、计算 cpu 使用率 = 1 - cpu 空闲率

```python
100 * (1 - sum (increase(node_cpu_seconds_total{mode="idle"}[5m]))  by (instance) / sum (increase(node_cpu_seconds_total[5m])) by (instance))
```

5、根据 irate() 函数，可以简化计算公式为：

```PYTHON
100 - (avg(irate(node_cpu_seconds_total{mode="idle"}[5m])) by (instance) * 100)
```

6、空闲内存剩余率：`(node_memory_MemFree_bytes+node_memory_Cached_bytes+node_memory_Buffers_bytes) / node_memory_MemTotal_bytes * 100`

7、内存使用率：`100 - (node_memory_MemFree_bytes+node_memory_Cached_bytes+node_memory_Buffers_bytes) / node_memory_MemTotal_bytes * 100`

8、磁盘使用率：`100 - (node_filesystem_free_bytes{mountpoint="/",fstype=~"ext4|xfs"} / node_filesystem_size_bytes{mountpoint="/",fstype=~"ext4|xfs"} * 100)`

##  0x04  Review（2024）

####  四大指标的意义及常用应用

以 http-cgi 为例，定义如下指标
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


####   实际应用：基于 alertmanager 的高级配置
如何使用 Conter 实现基于增量的告警策略？借助于 PromQL 编写适当的查询表达式，并将其配置为 Alertmanager 的告警规则来实现。下面使用 Prometheus Counter 监控 HTTP `5xx` 错误并在错误率超过阈值时触发告警，主要代码如下：

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

那么，使用如下 PromQL 来计算以过去 `5` 分钟内每分钟 HTTP `5xx` 错误的增量：

```python
rate(http_requests_total{code=~"5.."}[5m])
```

最后，在 Prometheus 配置文件中创建一个告警规则，当过去 `5` 分钟内 **每分钟的** HTTP `5xx` 错误率超过 `10` 时，触发告警：

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
      summary: "High error rate ({{$value}} errors/min)"
      description: "HTTP 5xx error rate is too high, affecting the application"
```

####  label 的意义
标签（label）是一种强大且灵活的元数据机制，用于对时间序列数据进行描述、区分和过滤。标签是键值对（key-value pair）形式的数据，可以附加到时间序列上，以提供有关数据来源和特性的详细信息。通过使用标签，Prometheus 可以更有效地查询和聚合数据，帮助开发人员和运维人员更好地理解和监控系统、应用程序和服务的性能

- 描述数据来源和特性：标签可以提供有关时间序列数据的详细描述，例如实例名称、服务名称、请求方法、状态码等
- 区分和过滤时间序列：标签允许你根据特定条件过滤和选择时间序列数据。例如，你可以使用标签查询特定服务或实例的性能指标，或者根据请求方法和状态码过滤 HTTP 请求数据
- 数据聚合：标签可以用于对数据进行分组和聚合，以便计算各种统计信息，如总和、平均值、最大值、最小值等

##  0x05 参考
-   [彻底理解 Prometheus 查询语法](https://blog.csdn.net/zhouwenjun0820/article/details/105823389)
-   [Prometheus 操作指南](https://github.com/yunlzheng/prometheus-book)
-   [理解时间序列](https://github.com/yunlzheng/prometheus-book/blob/master/promql/what-is-prometheus-metrics-and-labels.md)
-   [Prometheus Metrics 设计的最佳实践和应用实例，看这篇够了](https://cloud.tencent.com/developer/article/1639138)
- [PromQL 简明教程](https://www.kawabangga.com/posts/4408)
- [Prometheus 技术分享——prometheus 的函数与计算公式详解](https://zhuanlan.zhihu.com/p/595103670)
- [Prometheus 的函数和计算公式](https://blog.csdn.net/wc1695040842/article/details/107013799)
- [监控 metrics 系列 ---- Prometheus 入门](https://kingjcy.github.io/post/monitor/metrics/prometheus/prometheus/)
-   [PromQL 基本使用](https://songjiayang.gitbooks.io/prometheus/content/promql/summary.html)
-   [Prometheus 实战](https://songjiayang.gitbooks.io/prometheus/content/)