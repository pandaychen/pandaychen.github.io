---
layout:     post
title:      Kratos 源码分析：熔断器 Breaker
subtitle:   分析 Kratos 的熔断器实现
date:       2020-07-12
header-img: img/super-mario.jpg
author:     pandaychen
catalog:    true
tags:
    - Kratos
    - 熔断
---


##  0x00    前言
熔断器是为了当依赖的服务已经出现故障时，主动阻止对依赖服务的请求。保证自身服务的正常运行不受依赖服务影响，防止雪崩效应。本篇文章来分析下 Kratos 中的熔断器实现及应用。

#### Kratos 内置 breaker 的组件
一般情况下直接使用 Kratos 的组件时都自带了熔断逻辑，并且在提供了对应的 breaker 配置项。目前在 Kratos 内集成熔断器的组件有:
- RPC client: pkg/net/rpc/warden/client
- Mysql client：pkg/database/sql
- Tidb client：pkg/database/tidb
- Http client：pkg/net/http/blademaster

##	0x01	全局变量
在 [`breaker.go`](https://github.com/go-kratos/kratos/blob/master/pkg/net/netutil/breaker/breaker.go) 中定义了 `3` 个全局变量：
-	`_mu`
-	`_conf`
-	`_group`
```golang
var (
	_mu   sync.RWMutex
	_conf = &Config{
		Window:  xtime.Duration(3 * time.Second),
		Bucket:  10,
		Request: 100,

		// Percentage of failures must be lower than 33.33%
		K: 1.5,

		// Pattern: "",
	}
	_group = NewGroup(_conf)
)
```

####	熔断器状态定义
```golang
const (
	// StateOpen when circuit breaker open, request not allowed, after sleep
	// some duration, allow one single request for testing the health, if ok
	// then state reset to closed, if not continue the step.
	StateOpen int32 = iota
	// StateClosed when circuit breaker closed, request allowed, the breaker
	// calc the succeed ratio, if request num greater request setting and
	// ratio lower than the setting ratio, then reset state to open.
	StateClosed
	// StateHalfopen when circuit breaker open, after slepp some duration, allow
	// one request, but not state closed.
	StateHalfopen

	//_switchOn int32 = iota
	// _switchOff
)
```


##  0x01    breaker 的公共接口
熔断器 breaker 的公共部分封装 [在此](https://github.com/go-kratos/kratos/blob/master/pkg/net/netutil/breaker/breaker.go)，包含了熔断器的实现接口、熔断器组的定义，以及配置等等。


单个熔断器的接口定义如下：
-	`Allow` 方法：每次 RPC、接口调用时均会调用，用来判断是否在熔断状态，根据结果（`error`）来决定后续动作
-	`MarkSuccess`：每次 RPC 接口完成后，上报 Succ 状态（Kratos 中用 `RollingPolicy` 结构）
-	`MarkFailed`：每次 RPC 接口完成后，上报 Failed 状态

`MarkSuccess` 和 `MarkFailed` 都是用来完成熔断器状态检测的计算依据，所以针对何种错误下进行标记十分重要。

```golang
// Breaker is a CircuitBreaker pattern.
// FIXME on int32 atomic.LoadInt32(&b.on) == _switchOn
type Breaker interface {
	Allow() error
	MarkSuccess()
	MarkFailed()
}
```

熔断器的配置 Config：
```golang
type Config struct {
	SwitchOff bool // 熔断器开关, 默认关 false.

	K float64  // 触发熔断的错误率（K = 1 - 1 / 错误率）

	Window  xtime.Duration // 统计桶窗口时间
	Bucket  int  // 统计桶大小
	Request int64 // 触发熔断的最少请求数量（请求少于该值时不会触发熔断）
}
```

熔断器组的定义如下，注意 `brks` 是一个 map，以不通的 `key` 标识不同的熔断器，比如在 gRPC 中，我们可能对每个不同的 RPC 方法创建熔断器，那么使用 RPC 的名字作为 `key` 就非常合适。此外 `Breaker` 是 `interface{}` 类型，这里会被实例化为用户实现的熔断器类型，比如 Kratos 中的 `SREBreaker`：
```golang
// Group represents a class of CircuitBreaker and forms a namespace in which
// units of CircuitBreaker.
type Group struct {
	mu   sync.RWMutex		// 用来保护 brks 并发
	brks map[string]Breaker
	conf *Config
}
```

##	0x02  Group 公共接口定义
`newBreaker` 方法，生成一个 breaker 对象，其中 `newSRE` 是创建对应的熔断器对象：
```golang
// newBreaker new a breaker.
func newBreaker(c *Config) (b Breaker) {
	// factory
	return newSRE(c)
}
```

Breaker 中的 Group 和前文 [Kratos 源码分析（一）：Lazy Load Container](https://pandaychen.github.io/2020/06/06/LAZY-CONTAINER-GROUP/) 有点细微差别。Breaker 重新做了定义，和 Group 有关的操作如下：
-	`NewGroup`：创建一个熔断器组
-	`Group.Get`：
-	`Group.Reload`：
-	`Group.Go`

```golang
// NewGroup new a breaker group container, if conf nil use default conf.
func NewGroup(conf *Config) *Group {
	if conf == nil {
		_mu.RLock()
		conf = _conf
		_mu.RUnlock()
	} else {
		conf.fix()
	}
	// 构建 group 组
	return &Group{
		conf: conf,
		brks: make(map[string]Breaker),
	}
}
```

```golang
// Get get a breaker by a specified key, if breaker not exists then make a new one.
func (g *Group) Get(key string) Breaker {
	g.mu.RLock()
	brk, ok := g.brks[key]
	conf := g.conf
	g.mu.RUnlock()
	if ok {
		// 如果 key 对应的 Breaker 存在，则直接返回
		return brk
	}
	// 否则创建一个新的熔断器，并返回
	brk = newBreaker(conf)
	g.mu.Lock()
	if _, ok = g.brks[key]; !ok {
		g.brks[key] = brk
	}
	g.mu.Unlock()
	return brk
}
```

`Reload` 方法，重新（按照传入的 `conf` 配置）初始化熔断器组：
```golang
// Reload reload the group by specified config, this may let all inner breaker
// reset to a new one.
func (g *Group) Reload(conf *Config) {
	if conf == nil {
		return
	}
	conf.fix()
	g.mu.Lock()
	g.conf = conf
	g.brks = make(map[string]Breaker, len(g.brks))
	g.mu.Unlock()
}
```

`Go` 方法：
```golang
// Go runs your function while tracking the breaker state of group.
func (g *Group) Go(name string, run, fallback func() error) error {
	breaker := g.Get(name)
	if err := breaker.Allow(); err != nil {
		return fallback()
	}
	return run()
}
```

此外，Breaker 还暴露了一个使用默认熔断器（全局变量）配置的 `Go` 方法，应用程序可以直接调用，就像 hystix-Go 的 Go 方法那样：
```golang
// Go runs your function while tracking the breaker state of default group.
func Go(name string, run, fallback func() error) error {
	breaker := _group.Get(name)
	if err := breaker.Allow(); err != nil {
		return fallback()
	}
	return run()
}
```

##	0x03	SRE-Breaker 实现
[SRE-Breaker](https://github.com/go-kratos/kratos/blob/master/pkg/net/netutil/breaker/sre_breaker.go) 是 Breaker 的实现（的一种），按照 `Breaker` 定义的接口描述实现了具体的功能。

首先看下 `sreBreaker` 的定义，它使用了 `metric.RollingCounter` 作为统计结构，这也比较好理解，对于熔断器的判定指标，只需要关注一段时间内，正确响应和失败响应这两个指标的数量即可。另外，使用 `rand.Rand` 生成一个随机数，用来与 ** 客户端请求拒绝的概率 ** 做比较，前文 [Google SRE 弹性熔断算法实现分析](https://pandaychen.github.io/2020/05/10/A-GOOGLE-SRE-BREAKER/) 已做分析。

```golang
// sreBreaker is a sre CircuitBreaker pattern.
type sreBreaker struct {
	stat metric.RollingCounter
	r    *rand.Rand
	// rand.New(...) returns a non thread safe object
	randLock sync.Mutex

	k       float64		// 比率
	request int64

	state int32
}
```


####	初始化 SRE
通过 `newSRE` 来创建一个 `SreBreaker`：
```golang
func newSRE(c *Config) Breaker {
	counterOpts := metric.RollingCounterOpts{
		Size:           c.Bucket,		// 滑动窗口的桶数
		BucketDuration: time.Duration(int64(c.Window) / int64(c.Bucket)),
	}
	// 创建 NewRollingCounter
	stat := metric.NewRollingCounter(counterOpts)
	return &sreBreaker{
		stat: stat,
		r:    rand.New(rand.NewSource(time.Now().UnixNano())),

		request: c.Request,// 触发熔断的最少请求数量（请求少于该值时不会触发熔断）
		k:       c.K,
		state:   StateClosed,
	}
}
```

####	熔断状态上报
在 [理解 Kratos 的数据统计类型 Metrics（二）](https://pandaychen.github.io/2020/05/12/KRATOS-METRICS-ANALYSIS-2/#0x01----rollingcounter) 中分析过 `RollingCounter` 的 `Add` 方法，在 `SreBreaker` 中简单封装了此操作：
```golang
func (b *sreBreaker) MarkSuccess() {
	// 接口调用成功
	b.stat.Add(1)
}

func (b *sreBreaker) MarkFailed() {
	// NOTE: when client reject requets locally, continue add counter let the
	// drop ratio higher.
	// 接口调用失败
	b.stat.Add(0)
}
```


####	熔断判定算法
`Allow` 方法是 Breaker 判定的核心方法，在每次服务端执行真正的 RPC 请求前都需要调用 `Allow` 方法来进行熔断状态判定，`Allow` 方法的大致流程如下：
1.	通过 `summary()` 方法拿到当前滑动窗口 `metric.RollingCounter` 中统计的（滑动窗口）成功量 `success` 和请求总量 `total`
2.	根据 GoogleSRE 算法来进行熔断状态判定：
	-
	-

```golang
func (b *sreBreaker) Allow() error {
	success, total := b.summary()
	k := b.k * float64(success)		// 拿到 K∗accepts
	if log.V(5) {
		log.Info("breaker: request: %d, succee: %d, fail: %d", total, success, total-success)
	}
	// check overflow requests = K * success
	if total <b.request || float64(total) < k {
		// total <= K * success 时，关闭熔断；
		if atomic.LoadInt32(&b.state) == StateOpen {
			atomic.CompareAndSwapInt32(&b.state, StateOpen, StateClosed)
		}
		return nil
	}

	// 否则，熔断器开启（说明 total > K * success）
	if atomic.LoadInt32(&b.state) == StateClosed {
		atomic.CompareAndSwapInt32(&b.state, StateClosed, StateOpen)
	}

	// 根据 googleSRE 的算法计算拿到客户端请求拒绝的概率
	dr := math.Max(0, (float64(total)-k)/float64(total+1))

	// 概率判定，返回 true OR false
	drop := b.trueOnProba(dr)
	if log.V(5) {
		log.Info("breaker: drop ratio: %f, drop: %t", dr, drop)
	}
	if drop {
		return ecode.ServiceUnavailable
	}
	return nil
}
```

进行统计的方法 `summary` 实现如下，调用 `RollingCounter.Reduce` 方法，累加滑动窗口 Bucket 中的 `bucket.Count` 和 `bucket.Points`：
```golang
func (b *sreBreaker) summary() (success int64, total int64) {
	b.stat.Reduce(func(iterator metric.Iterator) float64 {
		for iterator.Next() {
			bucket := iterator.Bucket()
			total += bucket.Count
			for _, p := range bucket.Points {
				success += int64(p)
			}
		}
		return 0
	})
	return
}
```

而 `trueOnProba` 方法就是产生一个 `0-1` 之间的浮点数，和 `proba` 进行比较，间接实现了比较大小得出某个概率的方法，有个细节是 `b.r.Float64()` 是非线程安全的，所以加了 Lock：
```golang
func (b *sreBreaker) trueOnProba(proba float64) (truth bool) {
	b.randLock.Lock()
	truth = b.r.Float64() < proba
	b.randLock.Unlock()
	return
}
```

##	0x06	Warden 中的 Breaker 拦截器实现
Warden 中默认 gRPC 的 Client 端封装了 Breaker 熔断器机制，采用拦截器方式实现：
在 [`warden.Client`](https://github.com/go-kratos/kratos/blob/master/pkg/net/rpc/warden/client.go#L80) 中封装了 `breaker.Group`：
```golang
type ClientConfig struct {
	...
	Breaker                *breaker.Config	// 熔断器的配置
	Method                 map[string]*ClientConfig
	...
}

type Client struct {
	conf    *ClientConfig
	breaker *breaker.Group	// 封装了 breaker.Group
	mutex   sync.RWMutex
	...
}
```

初始化熔断器，使用 `SetConfig` 方法：
```golang
func (c *Client) SetConfig(conf *ClientConfig) (err error) {
	......
	c.mutex.Lock()
	c.conf = conf
	if c.breaker == nil {
		c.breaker = breaker.NewGroup(conf.Breaker)
	} else {
		c.breaker.Reload(conf.Breaker)
	}
	c.mutex.Unlock()
	return nil
}
```

客户端熔断器应用的核心逻辑：使用拦截器方式，在发起 RPC 调用前，检查熔断器的状态，在方法 [`handle`](https://github.com/go-kratos/kratos/blob/master/pkg/net/rpc/warden/client.go#L101) 的实现中：
```golang
// handle returns a new unary client interceptor for OpenTracing\Logging\LinkTimeout.
func (c *Client) handle() grpc.UnaryClientInterceptor {
	return func(ctx context.Context, method string, req, reply interface{}, cc *grpc.ClientConn, invoker grpc.UnaryInvoker, opts ...grpc.CallOption) (err error) {
		var (
			ok     bool
			t      trace.Trace
			gmd    metadata.MD
			conf   *ClientConfig
			cancel context.CancelFunc
			addr   string
			p      peer.Peer
		)
		var ec ecode.Codes = ecode.OK

		......
		// 根据 RPC 方法获取熔断器的配置
		brk := c.breaker.Get(method)
		// 检查熔断器状态
		if err = brk.Allow(); err != nil {
			// 注意：熔断器开启时错误，不能计入滑动窗口错误
			// 上报 metrics（熔断器错误计数）
			_metricClientReqCodeTotal.Inc(method, "breaker")
			return
		}
		// 根据错误结果上报熔断器结果状态
		defer onBreaker(brk, &err)

		......

		return
	}
}
```

而 `onBreaker` 方法是根据指定类型的错误，向熔断器的滑动窗口上报统计状态：
```golang
func onBreaker(breaker breaker.Breaker, err *error) {
	if err != nil && *err != nil {
		if  ecode.EqualError(ecode.ServerErr, *err) ||
			ecode.EqualError(ecode.ServiceUnavailable, *err) ||
			ecode.EqualError(ecode.Deadline, *err) ||
			ecode.EqualError(ecode.LimitExceed, *err) {
			breaker.MarkFailed()
			return

		}
	}
	breaker.MarkSuccess()
}
```

##	0x07	熔断器的一般使用
在其他的客户端场景，也可以使用熔断器来进行服务保护：
```golang
	// 初始化熔断器组
	// 一组熔断器公用同一个配置项，可从分组内取出单个熔断器使用。可用在比如 mysql 主从分离等场景。
	brkGroup := breaker.NewGroup(&breaker.Config{})
	// 为每一个连接指定一个 brekaker
	// 此处假设一个客户端连接对象实例为 conn
	//breakName 定义熔断器名称 一般可以使用连接地址
	breakName = conn.Addr
	conn.breaker = brkGroup.Get(breakName)

	// 在连接发出请求前判断熔断器状态
	if err = conn.breaker.Allow(); err != nil {
		return
	}

	// 连接执行成功或失败将结果告知 breaker
	if(respErr != nil){
		conn.breaker.MarkFailed()
	}else{
		conn.breaker.MarkSuccess()
	}
```

##  参考
-   [breaker.go](https://github.com/go-kratos/kratos/blob/master/pkg/net/netutil/breaker/breaker.go)
-   [sre_breaker.go](https://github.com/go-kratos/kratos/blob/master/pkg/net/netutil/breaker/sre_breaker.go)