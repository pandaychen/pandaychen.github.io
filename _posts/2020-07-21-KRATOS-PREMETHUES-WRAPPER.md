---
layout:     post
title:      Kratos 源码分析：Metrics 与 Prometheus 的集成与使用
subtitle:   分析 Kratos 的数据指标采集与使用
date:       2020-07-21
header-img: img/golang-horse-fly.png
author:     pandaychen
catalog:    true
tags:
    - Kratos
    - Prometheus
---


##  0x00    前言
前面已经分析过 Kratos 中的滑动窗口及基于滑动窗口实现的 RollingCounter、RollingGauge 实现，主要 [代码在此](https://github.com/go-kratos/kratos/tree/master/pkg/stat/metric)。
这篇文章主要分析如下几点内容：
1.  Kratos 中对 Prometheus 接口的集成及封装
2.  Warden 中使用 Metrics 做指标数据上报的拦截器封装
3.  Metrics 的自定义使用


主要引用了这个包 `github.com/prometheus/client_golang/prometheus` 来实现。

##  0x01    基础结构及全局变量

####    全局变量


####    基础结构 Metric
Metrics[定义](https://github.com/go-kratos/kratos/blob/master/pkg/stat/metric/metric.go) 如下，是最基础的结构，注意这句注释：
>   If the metric's type is PointGauge, RollingCounter, RollingGauge,
>   it returns the sum value within the window.

```golang
// Metric is a sample interface.
// Implementations of Metrics in metric package are Counter, Gauge,
// PointGauge, RollingCounter and RollingGauge.
type Metric interface {
	// Add adds the given value to the counter.
	Add(int64)
	// Value gets the current value.
	// If the metric's type is PointGauge, RollingCounter, RollingGauge,
	// it returns the sum value within the window.
	Value() int64
}
```

```golang
// Aggregation contains some common aggregation function.
// Each aggregation can compute summary statistics of window.
type Aggregation interface {
	// Min finds the min value within the window.
	Min() float64
	// Max finds the max value within the window.
	Max() float64
	// Avg computes average value within the window.
	Avg() float64
	// Sum computes sum value within the window.
	Sum() float64
}

// VectorOpts contains the common arguments for creating vec Metric..
type VectorOpts struct {
	Namespace string
	Subsystem string
	Name      string
	Help      string
	Labels    []string
}

const (
	_businessNamespace          = "business"
	_businessSubsystemCount     = "count"
	_businessSubSystemGauge     = "gauge"
	_businessSubSystemHistogram = "histogram"
)

var (
	_defaultBuckets = []float64{5, 10, 25, 50, 100, 250, 500}
)

// NewBusinessMetricCount business Metric count vec.
// name or labels should not be empty.
func NewBusinessMetricCount(name string, labels ...string) CounterVec {
	if name == "" || len(labels) == 0 {
		panic(errors.New("stat:metric business count metric name should not be empty or labels length should be greater than zero"))
	}
	return NewCounterVec(&CounterVecOpts{
		Namespace: _businessNamespace,
		Subsystem: _businessSubsystemCount,
		Name:      name,
		Labels:    labels,
		Help:      fmt.Sprintf("business metric count %s", name),
	})
}

// NewBusinessMetricGauge business Metric gauge vec.
// name or labels should not be empty.
func NewBusinessMetricGauge(name string, labels ...string) GaugeVec {
	if name == "" || len(labels) == 0 {
		panic(errors.New("stat:metric business gauge metric name should not be empty or labels length should be greater than zero"))
	}
	return NewGaugeVec(&GaugeVecOpts{
		Namespace: _businessNamespace,
		Subsystem: _businessSubSystemGauge,
		Name:      name,
		Labels:    labels,
		Help:      fmt.Sprintf("business metric gauge %s", name),
	})
}

// NewBusinessMetricHistogram business Metric histogram vec.
// name or labels should not be empty.
func NewBusinessMetricHistogram(name string, buckets []float64, labels ...string) HistogramVec {
	if name == "" || len(labels) == 0 {
		panic(errors.New("stat:metric business histogram metric name should not be empty or labels length should be greater than zero"))
	}
	if len(buckets) == 0 {
		buckets = _defaultBuckets
	}
	return NewHistogramVec(&HistogramVecOpts{
		Namespace: _businessNamespace,
		Subsystem: _businessSubSystemHistogram,
		Name:      name,
		Labels:    labels,
		Buckets:   buckets,
		Help:      fmt.Sprintf("business metric histogram %s", name),
	})
}
```

##  0x02	Gauge 封装

[](https://github.com/go-kratos/kratos/blob/master/pkg/stat/metric/gauge.go)，
```golang
// Gauge stores a numerical value that can be add arbitrarily.
type Gauge interface {
	Metric
	// Sets sets the value to the given number.
	Set(int64)
}

// GaugeOpts is an alias of Opts.
type GaugeOpts Opts

type gauge struct {
	val int64
}

// NewGauge creates a new Gauge based on the GaugeOpts.
func NewGauge(opts GaugeOpts) Gauge {
	return &gauge{}
}

func (g *gauge) Add(val int64) {
	atomic.AddInt64(&g.val, val)
}

func (g *gauge) Set(val int64) {
	old := atomic.LoadInt64(&g.val)
	atomic.CompareAndSwapInt64(&g.val, old, val)
}

func (g *gauge) Value() int64 {
	return atomic.LoadInt64(&g.val)
}

// GaugeVecOpts is an alias of VectorOpts.
type GaugeVecOpts VectorOpts

// GaugeVec gauge vec.
type GaugeVec interface {
	// Set sets the Gauge to an arbitrary value.
	Set(v float64, labels ...string)
	// Inc increments the Gauge by 1. Use Add to increment it by arbitrary
	// values.
	Inc(labels ...string)
	// Add adds the given value to the Gauge. (The value can be negative,
	// resulting in a decrease of the Gauge.)
	Add(v float64, labels ...string)
}

// gaugeVec gauge vec.
type promGaugeVec struct {
	gauge *prometheus.GaugeVec
}

// NewGaugeVec .
func NewGaugeVec(cfg *GaugeVecOpts) GaugeVec {
	if cfg == nil {
		return nil
	}
	vec := prometheus.NewGaugeVec(
		prometheus.GaugeOpts{
			Namespace: cfg.Namespace,
			Subsystem: cfg.Subsystem,
			Name:      cfg.Name,
			Help:      cfg.Help,
		}, cfg.Labels)
	prometheus.MustRegister(vec)
	return &promGaugeVec{
		gauge: vec,
	}
}

// Inc Inc increments the counter by 1. Use Add to increment it by arbitrary.
func (gauge *promGaugeVec) Inc(labels ...string) {
	gauge.gauge.WithLabelValues(labels...).Inc()
}

// Add Inc increments the counter by 1. Use Add to increment it by arbitrary.
func (gauge *promGaugeVec) Add(v float64, labels ...string) {
	gauge.gauge.WithLabelValues(labels...).Add(v)
}

// Set set the given value to the collection.
func (gauge *promGaugeVec) Set(v float64, labels ...string) {
	gauge.gauge.WithLabelValues(labels...).Set(v)
}
```

##  0x03    Counter 计数器

包含两部分，一部分是 `metric.Metrics` 封装的 Counter 结构（用于内部组件），另一部分是基于 `prometheus.CounterVec` 封装的计数器 Metrics（用于上报数据）：

####    Counter 结构及方法

`metric.Counter`[代码](https://github.com/go-kratos/kratos/blob/master/pkg/stat/metric/counter.go) 封装了 Prometheus 的 Counter 结构和方法：
`Counter` 是方法集合，包含了 `Metric`，`counter` 是具体结构，成员 `val int64` 用来存储计数器的值（仅自增）：
```golang
// Counter stores a numerical value that only ever goes up.
type Counter interface {
	Metric
}

type counter struct {
	val int64
}
```

`metric.counter` 提供了三个方法：
1.  `NewCounter`：初始化 `counter`
2.  `Add`：累加计数器
3.  `Value`：返回计数器的值

```golang
// NewCounter creates a new Counter based on the CounterOpts.
func NewCounter(opts CounterOpts) Counter {
	return &counter{}
}

func (c *counter) Add(val int64) {
	if val < 0 {
		panic(fmt.Errorf("stat/metric: cannot decrease in negative value. val: %d", val))
	}
	atomic.AddInt64(&c.val, val)
}

func (c *counter) Value() int64 {
	return atomic.LoadInt64(&c.val)
}
```

####    prometheus.CounterVec 封装及方法
Kratos 基于 `prometheus` 包封装了其 `prometheus.CounterVec` 的结构，`metric.promCounterVec`：
```golang
// CounterVecOpts is an alias of VectorOpts.
type CounterVecOpts VectorOpts

// CounterVec counter vec.
type CounterVec interface {
	// Inc increments the counter by 1. Use Add to increment it by arbitrary
	// non-negative values.
	Inc(labels ...string)
	// Add adds the given value to the counter. It panics if the value is <
	// 0.
	Add(v float64, labels ...string)
}

// counterVec counter vec.
type promCounterVec struct {
	counter *prometheus.CounterVec
}
```

提供了几个方法（注意方法中需要传入 `labels` 标签属性值）：
1.  `NewCounterVec`：初始化创建一个 `CounterVec`，注意传入的 `CounterOpts` 需要定义配置参数，如 `Namespace`、`Subsystem`、`Name` 及 `Help` 等
2.  `Inc`：封装调用 `prometheus.CounterVec` 的 `WithLabelValues(...).Inc` 方法
3.  `Add`：封装调用 `prometheus.CounterVec` 的 `WithLabelValues(...).Add` 方法

`NewCounterVec`，封装了 Prometheus 的初始化逻辑，返回 `metric.CounterVec` 对象：
```golang
// NewCounterVec .
func NewCounterVec(cfg *CounterVecOpts) CounterVec {
	if cfg == nil {
		return nil
	}
	vec := prometheus.NewCounterVec(
		prometheus.CounterOpts{
			Namespace: cfg.Namespace,
			Subsystem: cfg.Subsystem,
			Name:      cfg.Name,
			Help:      cfg.Help,
        }, cfg.Labels)
    // 注册 vec 到 Prometheus 中
	prometheus.MustRegister(vec)
	return &promCounterVec{
		counter: vec,
	}
}
```
`Inc` 和 `Add` 方法：
```golang
// Inc Inc increments the counter by 1. Use Add to increment it by arbitrary.
func (counter *promCounterVec) Inc(labels ...string) {
    // 封装方法
	counter.counter.WithLabelValues(labels...).Inc()
}

// Add Inc increments the counter by 1. Use Add to increment it by arbitrary.
func (counter *promCounterVec) Add(v float64, labels ...string) {
    // 封装方法
	counter.counter.WithLabelValues(labels...).Add(v)
}
```

##  0x04	Histogram 统计直方图
[Histogram 实现](https://github.com/go-kratos/kratos/blob/master/pkg/stat/metric/histogram.go)，初始化时需要传入 `Buckets   []float64` 数据：

```golang
// Timing adds a single observation to the histogram.
func (histogram *promHistogramVec) Observe(v int64, labels ...string) {
	histogram.histogram.WithLabelValues(labels...).Observe(float64(v))
}
```

至此，已经分析完成 Kratos 中对 Prometheus 的几种不同类型接口的封装，下面篇幅来看下实际中如何调用这些接口。

##  0x05	Warden 的 Metrics 拦截器
这里先看下 Warden 中是如何使用上述封装的 Metric 进行指标数据上报的。分两块：
1.  结构定义的部分 [在此](https://github.com/go-kratos/kratos/blob/master/pkg/net/rpc/warden/metrics.go)
2.  调用的部分 [在此](https://github.com/go-kratos/kratos/blob/master/pkg/net/rpc/warden/logging.go)


####    指标定义
Warden 中默认定义了几个监控指标，分为服务端和客户端：
-   `_metricServerReqDur`：服务端 RPC 处理延时，以 histogram 方式上报
-   `_metricServerReqCodeTotal`：服务端（按 code 类型）请求总数，以 counter 方式上报
-   `_metricClientReqDur`：客户端 RPC 调用延时，同样以 histogram 方式上报
-   `_metricClientReqCodeTotal`：客户端（按 code 类型）请求总数，以 counter 方式上报

代码如下：
```golang
// 默认的 namespace 名字
const (
	clientNamespace = "grpc_client"
	serverNamespace = "grpc_server"
)

var (
	_metricServerReqDur = metric.NewHistogramVec(&metric.HistogramVecOpts{
		Namespace: serverNamespace,
		Subsystem: "requests",
		Name:      "duration_ms",
		Help:      "grpc server requests duration(ms).",
		Labels:    []string{"method", "caller"},
		Buckets:   []float64{5, 10, 25, 50, 100, 250, 500, 1000},
	})
	_metricServerReqCodeTotal = metric.NewCounterVec(&metric.CounterVecOpts{
		Namespace: serverNamespace,
		Subsystem: "requests",
		Name:      "code_total",
		Help:      "grpc server requests code count.",
		Labels:    []string{"method", "caller", "code"},
	})
	_metricClientReqDur = metric.NewHistogramVec(&metric.HistogramVecOpts{
		Namespace: clientNamespace,
		Subsystem: "requests",
		Name:      "duration_ms",
		Help:      "grpc client requests duration(ms).",
		Labels:    []string{"method"},
		Buckets:   []float64{5, 10, 25, 50, 100, 250, 500, 1000},
	})
	_metricClientReqCodeTotal = metric.NewCounterVec(&metric.CounterVecOpts{
		Namespace: clientNamespace,
		Subsystem: "requests",
		Name:      "code_total",
		Help:      "grpc client requests code count.",
		Labels:    []string{"method", "code"},
	})
)
```

####    metric 方法调用
调用的方法 [在此](https://github.com/go-kratos/kratos/blob/master/pkg/net/rpc/warden/logging.go)，先看下客户端的拦截器：

```golang
func clientLogging(dialOptions ...grpc.DialOption) grpc.UnaryClientInterceptor {
	defaultFlag := extractLogDialOption(dialOptions)
	return func(ctx context.Context, method string, req, reply interface{}, cc *grpc.ClientConn, invoker grpc.UnaryInvoker, opts ...grpc.CallOption) error {
		logFlag := extractLogCallOption(opts) | defaultFlag

		startTime := time.Now()
		var peerInfo peer.Peer
		opts = append(opts, grpc.Peer(&peerInfo))

		// invoker requests
		err := invoker(ctx, method, req, reply, cc, opts...)

		// after request
		code := ecode.Cause(err).Code()
		duration := time.Since(startTime)
		// monitor
		_metricClientReqDur.Observe(int64(duration/time.Millisecond), method)
		_metricClientReqCodeTotal.Inc(method, strconv.Itoa(code))

        ......
		return err
	}
}

```

服务端拦截器的实现也是类似：
```golang
// serverLogging warden grpc logging
func serverLogging(logFlag int8) grpc.UnaryServerInterceptor {
	return func(ctx context.Context, req interface{}, info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (interface{}, error) {
		startTime := time.Now()
		caller := metadata.String(ctx, metadata.Caller)
		if caller == "" {
			caller = "no_user"
		}
		var remoteIP string
		if peerInfo, ok := peer.FromContext(ctx); ok {
			remoteIP = peerInfo.Addr.String()
		}
		var quota float64
		if deadline, ok := ctx.Deadline(); ok {
			quota = time.Until(deadline).Seconds()
		}

		// call server handler
		resp, err := handler(ctx, req)

		// after server response
		code := ecode.Cause(err).Code()
		duration := time.Since(startTime)
		// monitor
		_metricServerReqDur.Observe(int64(duration/time.Millisecond), info.FullMethod, caller)
		_metricServerReqCodeTotal.Inc(info.FullMethod, caller, strconv.Itoa(code))

        ......
		return resp, err
	}
}
```

##  0x06	参考
-	[Kratos Metrics](https://github.com/go-kratos/kratos/tree/master/pkg/stat/metric)