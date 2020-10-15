---
layout:     post
title:      Kratos 源码分析：理解 Kratos 的数据统计类型 Metrics（二）
subtitle:   分析 Kratos 框架中的 Metrics：LB 算法中的应用
date:       2020-05-12
author:     pandaychen
header-img:
catalog: true
category:   false
tags:
    - Kratos
---

##  0x00    前言
上一篇博客 [理解 Kratos 的数据统计类型 Metrics（一）](https://pandaychen.github.io/2020/04/12/KRATOS-METRICS-ANALYSIS/) 中，分析了 Kratos 的滑动窗口类 - `RollingPolicy` 的实现。这篇文章进一步看下基于 `RollingPolicy` 进一步封装的应用结构：`RollingCounter` 和 `RollingGauge`。这两个结构在 Kratos 项目中应用较为广泛。

##  0x01    RollingCounter
`RollingCounter` 就是基于滑动窗口的计数器。常用于 RPC、HTTP 调用成功率统计，在 [WRR-LB 算法实现](https://github.com/go-kratos/kratos/blob/master/pkg/net/rpc/warden/balancer/wrr/wrr.go#L56) 中，就应用了 `RollingCounter`。

```golang
type subConn struct {
	conn balancer.SubConn
	addr resolver.Address
	meta wmeta.MD

	err     metric.RollingCounter		//rpc 调用的错误率统计
	latency metric.RollingGauge			//rpc 调用的时延统计
	si      serverInfo
	// effective weight
	ewt int64
	// current weight
	cwt int64
	// last score
	score float64
}
```

`rollingCounter` 是 `RollingPolicy` 的上一层封装：

```golang
type rollingCounter struct {
	policy *RollingPolicy
}


type RollingCounter interface {
	Metric			// with prometheus
	Aggregation
	Timespan() int
	// Reduce applies the reduction function to all buckets within the window.
	Reduce(func(Iterator) float64) float64
}
```

`RollingCounter` 的 `Add` 方法，限制了计数器累加值不能为负数：

```golang
func (r *rollingCounter) Add(val int64) {
    // 计数器不能为负数
	if val < 0 {
		panic(fmt.Errorf("stat/metric: cannot decrease in value. val: %d", val))
	}
	r.policy.Add(float64(val))
}
```

`r.policy.Add`，回想上一篇博客分析 Window 的 `Add` 方法：向指定的偏移 offset 的 `0` 号位置累加值。图示如下：
```golang
// Add adds the given value to the latest point within bucket.
func (r *RollingPolicy) Add(val float64) {
	r.add(r.window.Add, val)
}
```

![image](https://wx1.sbimg.cn/2020/05/08/rollingcounter.png)

接开头，在 [WRR-LB](https://github.com/go-kratos/kratos/blob/master/pkg/net/rpc/warden/balancer/wrr/wrr.go#L158) 中，初始化的代码如下：
```golang
......
err: metric.NewRollingCounter(metric.RollingCounterOpts{
	Size:           10,			//window 大小为 10
	BucketDuration: time.Millisecond * 100,		// 每个窗口代表 100ms
}),
......
```

对 RPC 调用结果，计数器 [累加值的代码](https://github.com/go-kratos/kratos/blob/master/pkg/net/rpc/warden/balancer/wrr/wrr.go#L236) 如下，判断 RPC 请求的返回错误代码命中则执行 `conn.err.Add(1)`，反之则 `conn.err.Add(0)`，当然无论调用成功还是失败，`count` 的值都是要 `+1` 的。
```golang
ev := int64(0) // error value ,if error set 1
if di.Err != nil {
	if st, ok := status.FromError(di.Err); ok {
		// only counter the local grpc error, ignore any business error
		if st.Code() != codes.Unknown && st.Code() != codes.OK {
			ev = 1
		}
	}
}
conn.err.Add(ev)
```

最后，再看下 `subConn.err` 提供的 `Reduce()` 归约方法，它有两个返回值，`err` 代表滑动窗口中错误的总计数，`req` 代表滑动窗口中请求的总计数。那么基于滑动窗口计算的错误率就是：

$cs = 1- \frac{float64(errc)}{float64(req)}$

```golang
......
errc, req := conn.errSummary()
......
cs := 1 - (float64(errc) / float64(req))
......

func (c *subConn) errSummary() (err int64, req int64) {
	c.err.Reduce(func(iterator metric.Iterator) float64 {
		for iterator.Next() {
			bucket := iterator.Bucket()
			req += bucket.Count			// 累加 bucket 的 Count 值
			for _, p := range bucket.Points {
				err += int64(p)			// 累加 bucket 的 Err 值
			}
		}
		return 0
	})
	return
}
```

##  0x02    RollingGauge
`RollingGauge` 是基于滑动窗口的瞬时数据统计器。它和 `RollingCounter` 的最大不同是向 Bucket 的 Points 数组，追加数据（Points 数组的每个 index 代表了一种维度的 gauge）。`RollingGauge` 的结构如下：

```golang
type rollingGauge struct {
	policy *RollingPolicy
}

// RollingGauge represents a ring window based on time duration.
// e.g. [[1, 2], [1, 2, 3], [1,2, 3, 4]]
type RollingGauge interface {
	Metric
	Aggregation
	// Reduce applies the reduction function to all buckets within the window.
	Reduce(func(Iterator) float64) float64
}
```

`RollingGauge` 调用了 Window 的 `Append` 方法来完成存值：

```golang
func (r *rollingGauge) Add(val int64) {
	r.policy.Append(float64(val))
}
```

```golang
// Append appends the given points to the window.
func (r *RollingPolicy) Append(val float64) {
	r.add(r.window.Append, val)
}
```
![image](https://wx1.sbimg.cn/2020/05/08/rollinggauge.png)

此外，在前一节提到的 `subConn` 结构中，计算 RPC 调用平均时延就使用了 `RollingGauge` 结构，使用方式和 `RollingCounter` 差不多：
```golang
......
// 基于滑动窗口计算平均 latency
lagv, lagc := conn.latencySummary()
......

func (c *subConn) latencySummary() (latency float64, count int64) {
	c.latency.Reduce(func(iterator metric.Iterator) float64 {
		for iterator.Next() {
			bucket := iterator.Bucket()
			count += bucket.Count		// 累加 RPC 请求的总数
			for _, p := range bucket.Points {
				latency += p			// 累加所有 RPC 的调用时延
			}
		}
		return 0
	})

	// 计算平均时延
	return latency / float64(count), count
}
```

##	0x03	总结
本文分析了 Kratos 基于滑动窗口实现的统计数据结构：`RollingCounter` 和 `RollingGauge`。

##  0x04	参考
-   [rolling_gauge 的源码](https://github.com/go-kratos/kratos/blob/master/pkg/stat/metric/rolling_gauge.go)
-   [rolling_counter 的源码](https://github.com/go-kratos/kratos/blob/master/pkg/stat/metric/rolling_counter.go)