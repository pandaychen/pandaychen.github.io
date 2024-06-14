---
layout: post
title: 理解 Prometheus 的基本数据类型及应用（实践篇）
subtitle: Prometheus 及 PromQL 的一些实践
date: 2020-04-13
author: pandaychen
header-img: img/golang-horse-fly.png
catalog: true
category: false
tags:
  - Prometheus
  - Metrics
  - PromQL
---

## 0x00 前言
本文是对前一篇文章 [理解 Prometheus 的基本数据类型及应用（基础篇）](https://pandaychen.github.io/2020/04/11/PROMETHEUS-METRICS-INTRO/) 的实践以及深入理解，推荐如下几篇文章：

- [PromQL 简明教程](https://www.kawabangga.com/posts/4408)

##  0x01  再看四类指标
指标名称和标签集的每个唯一组合定义了一条时间序列，而每个时间戳和浮点数定义了一个系列中的样本（即一个数据点），Prometheus 其实只有两种数据类型。Gauge 和 Counter，而 Histogram、 Summary 本质上都是 Counter

- couter 的常用场景：网卡上发送和接收的字节数，应用程序中的错误数量
- gauge 的常用场景：CPU 和内存使用的指标，或者队列的大小


####  计数器（Counter）
Counter 类型指标被用于单调增加的测量结果，只增不减。唯一的例外是 Counter 重启，在这种情况下，counter 值会被重置为 `0`。Counter 的实际值本身并不是十分有用，通常，** 一个计数器的值经常被用来计算两个时间戳之间的 delta 或者随时间变化的速率 **；举例如下 counter `http_requests_total`：

```TEXT
# Counter 的一个典型用例是记录 API 调用次数，这是一个总是会增加的测量值
# 指标名称是 http_requests_total，它有一个名为 api 的 label，值为 add_product，Counter 的值为 4633433。这意味着自从上次服务启动或 Counter 重置以来，add_product 的 API 已经被调用了 4633433 次。按照惯例，Counter 类型的指标通常以_total 为后缀

# HELP http_requests_total Total number of http api requests
# TYPE http_requests_total counter
http_requests_total{api="add_product"} 4633433
```

单纯的 counter 绝对数字并没有提供多少信息，但配合 PromQL 的 `rate` 函数可以计算该 API 每秒收到的请求数（增长速率），如下：

```TEXT
# 查询计算了过去 5 分钟内每秒 add_product API 的平均请求数
rate(http_requests_total{api="add_product"}[5m])
```

又如，为了计算一段时期内 counter 的绝对变化（delta），可以利用 `increate` 函数，下面的表达式会返回过去 `5` 分钟内的总请求数的增量，这相当于用每秒的速率乘以间隔时间的秒数：

```python
increase(http_requests_total{api="add_product"}[5m])
rate(http_requests_total{api="add_product"}[5m]) * 5 * 60  #rate 结果 * 时间跨度
```

#### 仪表（Gauge）
Gauge 指标用于可以任意增加或减少的测量指标（从 gauge 视角来看，前后两个时间点的指标，单从数值看没有关联关系，可理解为离散的点），以主机的内存使用情况为例：

```python
# HELP node_memory_used_bytes Total memory used in the node in bytes
# TYPE node_memory_used_bytes gauge
node_memory_used_bytes{hostname="host1.domain.com"} 943348382
```
该指标的字面意思是在当前这个时间点测量时，节点 `host1.domain.com` 使用的内存约为 `900 MB`。该指标的值是有意义的，不需要任何额外的计算，因为它告诉该节点上消耗了多少内存；与 Counter 指标时不同，`rate` 和 `delta` 函数对 Gauge 意义不大。然而，计算特定时间序列的平均数（`avg_over_time`）、最大值（`max_over_time`）、最小值（`min_over_time`）或百分比（`quantile_over_time`）的函数经常与 Gauge 一起使用。如要计算过去 `10` 分钟内在 `host1.domain.com` 上使用的平均内存：

```python
avg_over_time(node_memory_used_bytes{hostname="host1.domain.com"}[10m])
```

####  直方图（Histogram）
Histogram 指标对于表示测量的分布很有用，经常被用来测量请求持续时间或响应大小。直方图将整个测量范围划分为一组区间，称为桶，并计算每个桶中有多少测量值。一个直方图指标包括几个部分：

- 一个包含测量次数的 Counter，指标名称使用 `_count` 后缀，参考 `http_request_duration_seconds_count`
- 一个包含所有测量值之和的 Counter，指标名称使用 `_sum` 后缀，参考 `http_request_duration_seconds_sum`
- 直方图桶（bucket）被暴露为一系列的 Counter，使用指标名称的后缀 `_bucket` 和表示桶的上限的 `le` label。Prometheus 中的桶是包含桶的边界的，即一个上限为 `N` 的桶（即 `le` label）包括所有数值小于或等于 `N` 的数据点，参考 `http_request_duration_seconds_bucket`

基于如上细化指标，可以计算出：

1、请求 QPS：`rate(<basename>_count)`，计算每秒的请求次数

2、平均耗时： `rate(<basename>_sum[1m]) / rate(<basename>_count[1m])`

3、分位数：假设需要计算 `90%` 的分位数，即 `histogram_quantile`[算法](https://github.com/prometheus/prometheus/blob/main/promql/quantile.go#L109) 如下：

按照 `le` 标签排序，查找第一个满足 `>= 90% * <basename>_count` 的 `le`（本例中为 `>= 0.9 * 27892 = 25103` 即 `le` 为 `1`），并记下面的变量值：

- `bucketStart`：`buckets[b-1].upperBound` 对应的 `le` 标签的值，即 `0.5`
- `bucketEnd`：`buckets[b].upperBound` 对应 `le` 标签的值，即 `1.0`
- `buckets[b-1].count` 与 `buckets[b].count`，分别对应 `24101` 与 `26351`
- `count`：`buckets[b].count - buckets[b-1].count`，即 `26351-24101=2250`
- `rank`：`q * observations - buckets[b-1].count`，即 `25103-24101=1002`

最终，公式为：`bucketStart + (bucketEnd-bucketStart)*float64(rank/count)`，即 `0.5 + 0.5*(1002/2250) = 0.72` ，在样本量更大的情况下将更加精确（例子在下面）

`histogram_quantile` 的参考代码如下：

```GO
func bucketQuantile(q float64, buckets buckets) (float64, bool, bool) {
	if math.IsNaN(q) {
		return math.NaN(), false, false
	}
	if q < 0 {
		return math.Inf(-1), false, false
	}
	if q > 1 {
		return math.Inf(+1), false, false
	}
	slices.SortFunc(buckets, func(a, b bucket) int {
		// We don't expect the bucket boundary to be a NaN.
		if a.upperBound < b.upperBound {
			return -1
		}
		if a.upperBound > b.upperBound {
			return +1
		}
		return 0
	})
	if !math.IsInf(buckets[len(buckets)-1].upperBound, +1) {
		return math.NaN(), false, false
	}

	buckets = coalesceBuckets(buckets)
	forcedMonotonic, fixedPrecision := ensureMonotonicAndIgnoreSmallDeltas(buckets, smallDeltaTolerance)

	if len(buckets) < 2 {
		return math.NaN(), false, false
	}
	observations := buckets[len(buckets)-1].count
	if observations == 0 {
		return math.NaN(), false, false
	}
	rank := q * observations
  // 寻找第一个大于 rank 的下标
	b := sort.Search(len(buckets)-1, func(i int) bool { return buckets[i].count >= rank })

	if b == len(buckets)-1 {
		return buckets[len(buckets)-2].upperBound, forcedMonotonic, fixedPrecision
	}
	if b == 0 && buckets[0].upperBound <= 0 {
		return buckets[0].upperBound, forcedMonotonic, fixedPrecision
	}
	var (
		bucketStart float64
		bucketEnd   = buckets[b].upperBound
		count       = buckets[b].count
	)
	if b > 0 {
		bucketStart = buckets[b-1].upperBound
		count -= buckets[b-1].count
		rank -= buckets[b-1].count
	}
	return bucketStart + (bucketEnd-bucketStart)*(rank/count), forcedMonotonic, fixedPrecision
}
```

测量运行在 `host1.domain.com` 实例上的 `add_product` API 的响应时间的 Histogram 指标可以表示为：

```python
# HELP http_request_duration_seconds Api requests response time in seconds
# TYPE http_request_duration_seconds histogram
http_request_duration_seconds_sum{api="add_product" instance="host1.domain.com"} 8953.332
http_request_duration_seconds_count{api="add_product" instance="host1.domain.com"} 27892
http_request_duration_seconds_bucket{api="add_product" instance="host1.domain.com" le="0"}
http_request_duration_seconds_bucket{api="add_product", instance="host1.domain.com", le="0.01"} 0
http_request_duration_seconds_bucket{api="add_product", instance="host1.domain.com", le="0.025"} 8
http_request_duration_seconds_bucket{api="add_product", instance="host1.domain.com", le="0.05"} 1672
http_request_duration_seconds_bucket{api="add_product", instance="host1.domain.com", le="0.1"} 8954
http_request_duration_seconds_bucket{api="add_product", instance="host1.domain.com", le="0.25"} 14251
http_request_duration_seconds_bucket{api="add_product", instance="host1.domain.com", le="0.5"} 24101
http_request_duration_seconds_bucket{api="add_product", instance="host1.domain.com", le="1"} 26351
http_request_duration_seconds_bucket{api="add_product", instance="host1.domain.com", le="2.5"} 27534
http_request_duration_seconds_bucket{api="add_product", instance="host1.domain.com", le="5"} 27814
http_request_duration_seconds_bucket{api="add_product", instance="host1.domain.com", le="10"} 27881
http_request_duration_seconds_bucket{api="add_product", instance="host1.domain.com", le="25"} 27890
http_request_duration_seconds_bucket{api="add_product", instance="host1.domain.com", le="+Inf"} 27892
```


本例包括 `sum`、`counter` 和 `12` 个桶，借助于此 `sum` 和 `counter` 可以用来计算一个测量值随时间变化的平均值。通过下式可以得到过去 `5` 分钟的平均请求响应时间（指定了 `label`）：

```python
rate(http_request_duration_seconds_sum{api="add_product", instance="host1.domain.com"}[5m]) / rate(http_request_duration_seconds_count{api="add_product", instance="host1.domain.com"}[5m])
```

又如，可以被用来计算各时间序列的平均数，计算出所有 API 和实例在过去 `5` 分钟内的平均请求响应时间（不加 label）：

```python
sum(rate(http_request_duration_seconds_sum[5m])) / sum(rate(http_request_duration_seconds_count[5m]))
```

利用 Histogram，也可以在查询时使用 `histogram_quantile` 函数计算单个时间序列以及多个时间序列的百分位（Prometheus 使用分位数 `0-1` 而不是百分位数），例如，要计算在 `host1.domain.com` 上运行的 `add_product` API 响应时间的第 `99` 百分位数以及所有 API 和实例的响应时间的第 `99` 个百分点‍（Histograms 进行汇总）

```python
histogram_quantile(0.99, rate(http_request_duration_seconds_bucket{api="add_product", instance="host1.domain.com"}[5m]))
histogram_quantile(0.99, sum by (le) (rate(http_request_duration_seconds_bucket[5m])))  #需要注意这里的 sum by (le)
```


####  汇总（Summary）
像直方图一样，Summary 指标对于测量请求持续时间和响应体大小很有用。一个 Summary 指标包括这些指标：

- 一个包含总测量次数的 Counter，指标名称使用 `_count` 后缀
- 一个包含所有测量值之和的 Counter。指标名称使用_sum 后缀。可以选择使用带有分位数标签的指标名称，来暴露一些测量值的分位数指标。由于你不希望这些量值是从应用程序运行的整个时间内测得的，Prometheus 客户端库通常会使用流式的分位值，这些分位值是在一个滑动的（通常是可配置的）时间窗口上计算得到的

Summary 通常用来在客户端直接计算出某个值（通常是请求持续时间或响应大小等）的分位数，然后直接上报到 Prometheus，通过 `Observe()` 函数，Summary 会上报如下三个时序指标：

- `<basename>{quantile="${val}"}`：某个分位数的值
- `<basename>_sum`：和 Histogram 的一致，为值的总和
- `<basename>_count`：为 `Observe()` 函数调用的次数

因此，通过 Summary 可以计算出如下数据：

1、QPS：`rate(<basename>_count)`

2、平均值： `rate(<basename>_sum[1m]) / rate(<basename>_count[1m])`

3、分位数：`<basename>{quantile="${val}"}`

例如，测量在 `host1.domain.com` 上运行的 `add_product` API 端点实例的响应时间的 Summary 指标可以表示为：

```PYTHON
# HELP http_request_duration_seconds Api requests response time in seconds
# TYPE http_request_duration_seconds summary
http_request_duration_seconds_sum{api="add_product" instance="host1.domain.com"} 8953.332 #所有值的总和
http_request_duration_seconds_count{api="add_product" instance="host1.domain.com"} 27892
http_request_duration_seconds{api="add_product" instance="host1.domain.com" quantile="0"}
http_request_duration_seconds{api="add_product" instance="host1.domain.com" quantile="0.5"} 0.232227334
http_request_duration_seconds{api="add_product" instance="host1.domain.com" quantile="0.90"} 0.821139321
http_request_duration_seconds{api="add_product" instance="host1.domain.com" quantile="0.95"} 1.528948804
http_request_duration_seconds{api="add_product" instance="host1.domain.com" quantile="0.99"} 2.829188272
http_request_duration_seconds{api="add_product" instance="host1.domain.com" quantile="1"} 34.283829292
```

上例包括总和和计数以及五个分位数。分位数 `0` 相当于最小值，分位数 `1` 相当于最大值。分位数 `0.5` 是中位数，分位数 `0.90`、`0.95` 和 `0.99` 相当于在 `host1.domain.com` 上运行的 `add_product` API 后端响应时间的第 `90`、`95` 和 `99` 个百分位。像 Histogram 一样，Summary 指标包括总和和计数，可用于计算随时间的平均值以及不同时间序列的平均值。Summary 提供了比 Histogram 更精确的百分位计算结果，但这些百分位有三个主要缺点：

1.  首先，客户端计算百分位是很昂贵的。这是因为客户端库必须保持一个有序的数据点列表，以进行这种计算。在 Prometheus SDK 中的实现限制了内存中保留和排序的数据点的数量，这降低了准确性以换取效率的提高（并非所有的 Prometheus 客户端库都支持汇总指标中的量值）
2.  第二，你要查询的量值必须由客户端预先定义。只有那些已经提供了指标的量值才能通过查询返回。没有办法在查询时计算其他百分位。增加一个新的百分位指标需要修改代码，该指标才可以被使用
3.  无法把多个 Summary 指标进行聚合计算。在本例中若 `add_product` 的 API 后端运行在 `10` 个主机上，在这些服务之前有一个负载均衡器。没有任何聚合函数可以用来计算 `add_product` API 接口在所有请求中响应时间的第 `99` 百分位数，无论这些请求被发送到哪个后端实例上。只能看到每个主机的第 `99` 个百分点。同样也只能知道某个接口，比如 `add_product` API 端点的（在某个实例上的）第 `99` 百分位数，而无法对不同的接口进行聚合

##  0x02  promql 的典型应用：引入

本小节介绍下 PromQL 语言的基础概念，下图表示了 Metric 在 Prometheus 中的样子（假设有如下 `5` 张网卡的流量 counter 指标 `node_network_receive_packets_total`）：

![selector]()

![grafana-1]()

直接在 Grafana 中使用 `node_network_receive_packets_total` 来画图会得到 `5` 条线，不过此图意义不大且无法看到指标变化趋势。因为 Counter 只增加，可以认为是该服务器自存在以来收到的所有的包的数量。Metric 可以通过 `label` 来进行选择，比如 `node_network_receive_packets_total{device="bond0"}` 就会只查询到 `bond0` 的数据（绘制 `bond0` 这个 device 的曲线）, 当然也支持正则表达式，可以通过 `node_network_receive_packets_total{device=~"en.*"}` 绘制 `en0` 和 `en2` 的曲线。其实，metric name 也是一个 label， 所以 `node_network_receive_packets_total{device="bond0"}` 本质上是 `{__name__="node_network_receive_packets_total", device="bond0"}` 。但是因为 metric name 基本上是必用的 label，一般用第一种写法

但实际上，如果使用下面的查询语句，将会仅仅得到一个数字，而不是整个 metric 的历史数据（`node_network_receive_packets_total{device=~"en.*"}` 得到的是下图中黄色的部分（即 Instant Vector），只查询到 metric 的在某个时间点（默认是当前时间）的值。

![]()

####	Instant vector selectors
这里描述的 Instant Vector 还是 Range Vector, ** 指的是 PromQL 函数的入参和返回值的类型 **，Instant Vector 就是当前的值，假如查询的时间点是 `t`，那么查询会返回距离 `t` 时间点最近的一个值

![Instant vector]()

####	Range vector selectors
![Range vector]()

Range Vector 返回的是一个 range 的数据。Range 的表示方法是 `[1m]`，表示 `1` 分钟的数据，假如对 Prometheus 的采集配置是每 `10s` 采集一次，那么 `1m` 内就会有采集 `6` 次，就会有 `6` 个数据点，使用 `node_network_receive_packets_total{device=~"en.*"}[1m]` 查询的话，就可以得到以下的数据：两个 metric 的最后的 `6` 个数据点，如下图：

![Range vector selectors](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/metrics/prometheus/Prometheus-basic/select-en-range.png)


####	一个插曲
思考一个问题，在 Grafana 中画出来一个 Metric 的图标，需要查询结果是一个 Instant Vector，还是 Range Vector 呢？答案是 Instant Vector

####	常用操作


####	核心：rate 计算的本质
通常 counter 都会与 `rate` 函数结合计算增量，如下面的计算 `node_network_receive_packets_total` 在 `1m` 的时间跨度增量，即查询每秒收到的 packet 数量：

```python
rate(node_network_receive_packets_total{device=~"en.*"}[1m])
```

先来看一个时间点的计算，假如计算 `t` 时间点的每秒 packet 数量，`rate` 函数用这段时间（`[1m]`）的总 packet 数量，除以时间跨度 `[1m]`，就得到了一个 "平均值"，以此作为曲线来绘制，通过这种方法就得到了一个点的数据。如下图：

![counter-rate-1](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/metrics/prometheus/Prometheus-basic/select-en-range-1.png)


然后，对之前的每一个点，都以此法进行计算，就得到了一个 `packet/s` 的曲线（最长的那条是原始的数据，黄色的表示 `rate` 对于每一个点的计算过程，蓝色的框为最终的绘制的点），所以这个 PromQL 查询最终得到的数据点是：`<2.2, 1.96, 2.31, 2, 1.71>` （即蓝色的点构成的序列）

![counter-rate-2](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/metrics/prometheus/Prometheus-basic/rate-chart-1.png)

因为指定的 label 有两个选中的 metric，分别是 `en0` 和 `en2`，所以 `rate` 会分别计算两条曲线，就得到下面的 Chart，有两条线

![flow](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/metrics/prometheus/Prometheus-basic/network-rate-query.png)

这样的指标在笔者现网中也是极为常用的

####	rate VS irate
上节分析了 `rate` 的实现，对比 `irate` 则是计算的最后两个点之间的差值，如下图，注意对比与 `rate` 方法的不同：

![irate](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/metrics/prometheus/Prometheus-basic/irate-chart-.png)

`irate` 函数因为只用最后两个点的差值来计算，会比 `rate` 平均值的方法得到的结果，变化更加剧烈，更能反映当时的情况。那既然是使用最后两个点计算，这里又为什么需要时间跨度 `[1m]` 参数？这个 `[1m]` 不是用来计算的，是用来限制找 `t-2` 个点的时间的，比如，如果中间丢了很多数据，那么显然这个点的计算会很不准确，`irate` 在计算的时候会最多向前在 `[1m]` 找点，如果超过 `[1m]` 没有找到数据点，这个点的计算就放弃了。`irate(node_network_receive_packets_total{device=~"en.*"}[1m])` 的指标图更新如下：

![irate](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/metrics/prometheus/Prometheus-basic/network-irate-query.png)

对比与 `rate`，可以看到后者变化更加剧烈了，比较适合于 CPU，network 这种资源的变化，使用 `irate` 更加有意义

####	increase
`increase` 的计算方式是 `end - start`（没有除）计算的是每分钟的增量

小结下，`rate`/`irate`/`increase` 函数接受的都是 Range Vector，返回的是 Instant Vector，另外需要 [注意](https://www.robustperception.io/what-range-should-i-use-with-rate/) 的是，`increase` 和 `rate` 的 range 内必须要有至少 `4` 个数据点

####	使用函数的顺序问题
计算如下公式 `P99` 的时候，要注意函数顺序（`rate`、`sum` 与 `histogram_quantile` 的次序不能交换）：

```python
histogram_quantile(0.99,
    sum by (le)
    (rate(http_request_duration_seconds_bucket[10m]))
)
```

首先，Histogram 是一个 Counter，所以要使用 `rate` 先处理，然后根据 `le` 将 labels 使用 `sum` 合起来，最后使用 `histogram_quantile` 来计算。这三个函数的顺序是不能调换的，必须是先 `rate` 再 `sum`，最后 `histogram_quantile`

-	`rate` 必须在 `sum` 之前。前面提到过 Prometheus 支持在 Counter 的数据有下降之后自动处理的，比如服务器重启了，metric 重新从 `0` 开始。这个其实不是在存储的时候做的，比如应用暴露的 metric 就是从 `2033` 变成 `0` 了，那么 Prometheus 就会存储 `0`。 但是在计算 `rate` 的时候，就会识别出来这个下降。但是 `sum` 不会，所以如果先 `sum` 再 `rate`，曲线就会出现非常大的波动（通过 `rate` 尽力排除掉疑似 `0` 这种对结果的干扰）
-	`histogram_quantile` 必须在最后，由于 `histogram_quantile` 计算的结果是近似值，去聚合（无论是 `sum` 还是 `max` 还是 `avg`）这个值都是没有意义的，可以参考上述 `histogram_quantile` 的分析逻辑

####	常用


##  0x03  promql 的典型应用：实践

##  0x04  小结

####  summary vs Histogram
在大多数情况下，直方图是首选，因为它更灵活，并允许汇总百分位数。在不需要百分位数而只需要平均数的情况下，或者在需要非常精确的百分位数的情况下，汇总是有用的。例如，在履行关键系统的合约责任的情况下。


Histogram
客户端性能消耗小，服务端查询分位数时消耗大。
可以在查询期间自由计算各种不同的分位数。
分位数的精度无法保证，其精确度受桶的配置、数据分布、数据量大小情况影响。
可聚合，可以计算全局分位数。
客户端兼容性好。
Summary
客户端性能消耗大（因为分位数计算发生在客户端），服务端查询分位数时消耗小。
只能查询客户端上报的哪些分位数。
分位数的精度可以得到保证，精度会影响客户端的消耗。
不可聚合，无法计算全局分位数（因此不支持多实例，平行扩展的 http 服务）。
客户端兼容性不好。
综上所述，大多数场景使用 Histogram 更为灵活



####  PromQL 要先 rate() 再 sum()，不能 sum() 完再 rate()
这背后与 rate() 的实现方式有关，rate() 在设计上假定对应的指标是一个 Counter，也就是只有 incr(增加) 和 reset(归 0) 两种行为。而做了 sum() 或其他聚合之后，得到的就不再是一个 Counter 了，举个例子，比如 sum() 的计算对象中有一个归 0 了，那整体的和会下降，而不是归零，这会影响 rate() 中判断 reset(归 0) 的逻辑，从而导致错误的结果。写 PromQL 时这个坑容易避免，但碰到 Recording Rule 就不那么容易了，因为不去看配置的话大家也想不到 new_metric 是怎么来的。

Recording Rule 规则：一步到位，直接算出需要的值，避免算出一个中间结果再拿去做聚合。

```python
rate(http_requests_total{job="node"}[5m])
sum by (job)(rate(http_requests_total{job="node"}[5m]))  # This is okay
rate(sum by (job)(http_requests_total{job="node"})[5m])  # Don't do this
```

可以参考此文 [Rate then sum, never sum then rate](https://www.robustperception.io/rate-then-sum-never-sum-then-rate/)



## 0x07 参考

- [一文带你了解 Prometheus](https://cloud.tencent.com/developer/article/1999843)
- [一文搞懂 Prometheus 的直方图](https://juejin.im/post/5d492d1d5188251dff55b0b5)
- [如何区分 prometheus 中 Histogram 和 Summary 类型的 metrics？](https://www.cnblogs.com/aguncn/p/9920545.html)
- [Metrics 设计：teleport](https://goteleport.com/teleport/docs/metrics-logs-reference/)
- [Lock-free Observations for Prometheus Histograms](https://grafana.com/blog/2020/01/08/lock-free-observations-for-prometheus-histograms/)
- [golang API](https://godoc.org/github.com/prometheus/client_golang/prometheus)
-  [HISTOGRAMS AND SUMMARIES](https://prometheus.io/docs/practices/histograms/)
-  [prometheus 的内置函数](https://prometheus.io/docs/prometheus/latest/querying/functions/)
- [Prometheus 的九个坑](https://www.jianshu.com/p/c99375ef40aa)
- [PromQL 简明教程](https://www.kawabangga.com/posts/4408)
- [详解 Prometheus 四种指标类型](https://www.cnblogs.com/88223100/p/Explain-four-indicator-types-of-Prometheus.html)
- [Histogram and Summary](https://hulining.gitbook.io/prometheus/practices/histograms)
- [What is the difference between increase and delta in PromQL?](https://stackoverflow.com/questions/74991963/what-is-the-difference-between-increase-and-delta-in-promql)
- [P99 是如何计算的](https://www.kawabangga.com/posts/4284)
- [Understanding Prometheus Range Vectors](https://satyanash.net/software/2021/01/04/understanding-prometheus-range-vectors.html)

转载请注明出处，本文采用 [CC4.0](http://creativecommons.org/licenses/by-nc-nd/4.0/) 协议授权
