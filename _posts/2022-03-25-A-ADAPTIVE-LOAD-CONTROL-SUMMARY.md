---
layout: post
title: 微服务项目中的自适应技术（adaptive）分析与应用（TODO）
subtitle: 分析 go-zero 框架中自适应技术的运用
date: 2022-03-25
header-img: img/super-mario.jpg
author: pandaychen
catalog: true
tags:
  - 微服务框架
  - 自适应技术
---

## 0x00 前言
本篇文章分析下自适应技术在微服务领域的实践，以 go-zero[项目](https://go-zero.dev/) 为例，此项目非常值得借鉴。

##  0x01 背景
自适应解决了什么问题呢？以 circuitbreaker[熔断器](https://resilience4j.readme.io/docs/circuitbreaker) 而言，可选的配置参数非常多，和系统预期吞吐 / qps 的经验值都会有关系，配置合适的参数是一件很麻烦的事情。所以，通过自适应算法能让我们尽量少关注参数，只要简单配置就能满足大部分场景。

## 0x02   自适应的负载均衡实现
自适应的负载均衡，意为自动的选择指标最优的节点进行请求（如负载低、延时低等），负载低考虑 CPU / 内存，延时主要指接口响应；此外，要求可以动态发现后端节点以及隔离故障节点。从代码 [实现](https://github.com/zeromicro/go-zero/blob/master/zrpc/internal/balancer/p2c/p2c.go) 看，go-zero 采用了 P2C 算法 + EWMA 来实现自适应的 LB 机制。

####  基础
- P2C：一般来说，在多个节点中随机选择 `2` 个，然后再此 `2` 中选择一个最优
- EWMA： 指数移动加权平均法，其意义在于只需要保存最近一次的数值（如接口延时），利用加权因子来预估时间区间的平均值，算法的具体含义可参考 [此文](https://www.cnblogs.com/jiangxinyang/p/9705198.html)。EWMA 是指各数值的加权系数随时间呈指数递减，**越靠近当前时刻的数值加权系数就越大**，体现了最近一段时间内的平均值。

####  EWMA 的意义
为啥要是用 EWMA 呢？试想一下，当客户端发起请求时，实际上只能通过历史的数据（如上 `N` 次的延迟，上 `N` 次的服务端负载数据）来 "预测" 本次请求的延迟，所以 EWNA 算法是比较契合的。

1、公式 <br>

$$
V_{t} = \beta  \times V_{t-1} + (1- \beta) \times \theta_{t}
$$

-	$V_{t}$ ：代表的是第 `t` 次请求的 EWMA 值
-	$V_{t-1}$: 代表的是第 `t-1` 次请求的 EWMA 值
-	`β`: 常量，go-zero 使用牛顿冷却定律中的衰减函数模型计算

1.	相较于普通的计算平均值算法，EWMA 算法不需要保存过去所有的数值，计算量和存储都显著减少（只需要保存最新一次的数值即可）
2.	传统的计算平均值算法对网络耗时不敏感，而 EWMA 算法可以通过请求频繁来调节 `β`，进而迅速监控到网络毛刺或更多的体现整体平均值：
	-	当请求较为频繁时，说明节点网络负载升高了，我们想监测到此时节点处理请求的耗时 (侧面反映了节点的负载情况), 我们就相应的调小 `β`。`β` 越小，EWMA 值 就越接近本次耗时，进而迅速监测到网络毛刺；
	-	当请求较为不频繁时，我们就相对的调大 `β` 值，此时计算出来的 EWMA 值越接近平均值


####  β 定义
1、计算公式 <br>

$$
\beta =  {1 \over e^{k \times \Delta_{t}}}
$$

如上面的表达式，go-zero 采用的是 **牛顿冷却定律中的衰减函数模型** 计算 EWMA 算法中的 `β` 值（ `Δt` 为两次请求的间隔），对应的代码为：


```golang
const (
	decayTime = int64(time.Second * 10) // default value from finagle
)

func (p *p2cPicker) buildDoneFunc(c *subConn) func(info balancer.DoneInfo) {
	start := int64(timex.Now())
	return func(info balancer.DoneInfo) {
		//.......
		now := timex.Now()

    // 获取上一次请求的时间
		last := atomic.SwapInt64(&c.last, int64(now))
		td := int64(now) - last
		if td < 0 {
			td = 0
		}

    // 计算 belta 值，注意 td 为负
		w := math.Exp(float64(-td) / float64(decayTime))
    //.......
}
```

####  核心代码分析
基于 gRPC balancer 的经验，分两块：
1.	[Builder](https://github.com/zeromicro/go-zero/blob/master/zrpc/internal/balancer/p2c/p2c.go#L42)：每次有后端节点上下线时会调用，重新初始化 `[]*subConn`
2.	[Picker](https://github.com/zeromicro/go-zero/blob/master/zrpc/internal/balancer/p2c/p2c.go#L75)：根据负载均衡算法选择 `1` 个节点进行请求

######	封装的 balancer.SubConn 结构
一般要实现自己的 LB 算法，通常要把 `balancer.SubConn` 结构二次封装，加入 EMWA 值、权重、并发量等一些因子，go-zero 增加了如下因子：

-	`lag`：保存 EWMA 值
-	`inflight`：保存当前节点正在并发的请求总数
-	`success`：标识一段时间内此连接的健康状态
-	`requests`：请求总数
-	`last`：上一次请求耗时, 用于计算 EWMA 值
-	`pick`：该连接上一次被选中的时间戳

```golang
type subConn struct {
     addr     resolver.Address
     conn     balancer.SubConn

     lag      uint64 // 用来保存 EWMA 值
     inflight int64  // 用在保存当前节点正在处理的请求总数
     success  uint64 // 用来标识一段时间内此连接的健康状态
     requests int64  // 用来保存请求总数
     last     int64  // 用来保存上一次请求耗时, 用于计算 EWMA 值
     pick     int64  // 保存上一次被选中的时间点
}
```

1、load 方法 <br>
`load` 方法预估（计算）了 **节点的负载情况**，是通过下面公式实现：`load = ewma * inflight`，这里有点意思：

>	ewma 相当于平均请求耗时
>	inflight 是当前节点正在处理请求的数量

上面 `2` 个因子相乘大致计算出了当前节点的网络负载。

```golang
func (c *subConn) load() int64 {
	// 通过 EWMA 计算节点的负载情况； 加 1 是为了避免为 0 的情况
	lag := int64(math.Sqrt(float64(atomic.LoadUint64(&c.lag) + 1)))
	load := lag * (atomic.LoadInt64(&c.inflight) + 1)
	if load == 0 {
		// penalty：初始化没有数据时的惩罚值，默认为 1e9 * 250
		return penalty
	}

	return load
}
```

2、healthy 方法 <br>
`throttleSuccess` 为常量，`c.success` 初始值为 `throttleSuccess` 的 `2` 倍
```golang
func (c *subConn) healthy() bool {
	return atomic.LoadUint64(&c.success) > throttleSuccess
}
```

######	Builder
gRPC balancer 在后端节点有更新的时候会调用 `Build` 方法，传入所有节点信息 `info base.PickerBuildInfo`，使用 `p2cPicker.conns` 保存并（重新）初始化所有的节点信息：
```golang
type p2cPicker struct {
   conns []*subConn  // 保存所有服务的节点信息
   r     *rand.Rand
   stamp *syncx.AtomicDuration
   lock  sync.Mutex
}
```

```golang
//每次有后端节点新增/下线都会触发初始化
func (b *p2cPickerBuilder) Build(info base.PickerBuildInfo) balancer.Picker {
	readySCs := info.ReadySCs
	if len(readySCs) == 0 {
		return base.NewErrPicker(balancer.ErrNoSubConnAvailable)
	}

	var conns []*subConn
	for conn, connInfo := range readySCs {
		conns = append(conns, &subConn{
			addr:    connInfo.Address,
			conn:    conn,
			success: initSuccess,
		})
	}

	return &p2cPicker{
		conns: conns,
		r:     rand.New(rand.NewSource(time.Now().UnixNano())),
		stamp: syncx.NewAtomicDuration(),
	}
}
```

######	Picker
在 Pick 方法里实现了 P2C [算法](https://github.com/zeromicro/go-zero/blob/master/zrpc/internal/balancer/p2c/p2c.go#L75)，挑选合适的节点，并通过节点的 EWMA 值计算负载情况，返回负载低的节点供 gRPC 使用。go-zero 是使用的下面的 Pick 逻辑：
1.	多选二，基于 P2C 算法
2.	二再选一，基于 EWMA 负载低的原则

1、Pick 方法 <br>
```golang
func (p *p2cPicker) Pick(info balancer.PickInfo) (balancer.PickResult, error) {
	p.lock.Lock()
	defer p.lock.Unlock()

	var chosen *subConn
	switch len(p.conns) {
	case 0:
		// 无节点
		return emptyPickResult, balancer.ErrNoSubConnAvailable
	case 1:
		//only one，直接返回
		chosen = p.choose(p.conns[0], nil)
	case 2:
		// only two，直接进入 choose，通过 EWMA 值 计算负载，并返回负载低的节点返回供 gRPC 使用
		chosen = p.choose(p.conns[0], p.conns[1])
	default:
		// 超过 3 个节点，P2C 选择两个
		var node1, node2 *subConn
		for i := 0; i < pickTimes; i++ {
			a := p.r.Intn(len(p.conns))
			b := p.r.Intn(len(p.conns) - 1)
			if b >= a {
				b++
			}
			node1 = p.conns[a]
			node2 = p.conns[b]
			// 如果这次选择的节点达到了健康要求, 就中断选择
			if node1.healthy() && node2.healthy() {
				break
			}
		}
		 // 比较两个节点的负载情况，选择负载低的
		chosen = p.choose(node1, node2)
	}

	atomic.AddInt64(&chosen.inflight, 1)
	atomic.AddInt64(&chosen.requests, 1)

	return balancer.PickResult{
		SubConn: chosen.conn,
		Done:    p.buildDoneFunc(chosen),	// 请求结束，传入被选中的节点，更新此节点的 EWMA 等信息
	}, nil
}
```

2、choose 方法 <br>
`choose` 方法会调用 `load` 方法来计算节点负载
```golang
func (p *p2cPicker) choose(c1, c2 *subConn) *subConn {
	start := int64(timex.Now())
	if c2 == nil {
		atomic.StoreInt64(&c1.pick, start)
		return c1
	}

  // 如果 c1 指向 conn 的负载比 c2 指向 conn 的负载高，那么让 c1 指向负载低的，c2 指向高的
	if c1.load()> c2.load() {
		c1, c2 = c2, c1
	}

	pick := atomic.LoadInt64(&c2.pick)
	if start-pick > forcePick && atomic.CompareAndSwapInt64(&c2.pick, pick, start) {
		return c2
	}

	//pick c1，更新 pick 时间
	atomic.StoreInt64(&c1.pick, start)
	return c1
}
```

注意上面的这段代码的用意：如果选中的节点 `c2`（相对的高负载），在 `forcePick` 期间内没有被选中一次，那么强制选择一次。这里是利用强制的机会，来触发成功率、延迟的衰减，不然可能会导致此节点永远不会被选中，比较巧妙的设计。这里还使用了 `CompareAndSwapInt64` 方法，即原子锁保证并发安全，仅放行 `1` 次。
```golang
{
	//......
	pick := atomic.LoadInt64(&c2.pick)
	if start-pick > forcePick && atomic.CompareAndSwapInt64(&c2.pick, pick, start) {
		return c2
	}
	//......
}
```

3、`buildDoneFunc` 方法 <br>
在请求结束的时候 gRPC 会调用 `PickResult.Done` 方法，在此实现了本次请求耗时等信息的存储，并计算出了 EWMA 值保存了起来，供下次请求时计算负载等情况的使用：
-	被选中节点正在处理请求的总数减 `1`
-	保存处理请求结束的时间点，用于计算距离上次节点处理请求的差值，并算出 EWMA 中的 `β` 值
-	通过 EWMA 算法计算本次请求耗时（延迟），并保存到节点的 `lag` 属性里
-	通过 EWMA 算法，更新计算节点的健康状态保存到节点的 `success` 属性中

```golang
func (p *p2cPicker) buildDoneFunc(c *subConn) func(info balancer.DoneInfo) {
	start := int64(timex.Now())
	return func(info balancer.DoneInfo) {
		// 请求结束，把节点正在处理请求的总数（并发数）减 1
		atomic.AddInt64(&c.inflight, -1)
		now := timex.Now()

		// 保存处理请求结束的时间点，返回之前的值 c.last
		last := atomic.SwapInt64(&c.last, int64(now))
		td := int64(now) - last
		if td < 0 {
			td = 0
		}
		 // 用牛顿冷却定律中的衰减函数模型计算 EWMA 算法中的β值
		w := math.Exp(float64(-td) / float64(decayTime))	//w 即为 EMWA 的β值
		lag := int64(now) - start
		if lag < 0 {
			lag = 0
		}
		 // 保存本次请求的耗时，并返回上次的耗时，用于 ewma 计算
		olag := atomic.LoadUint64(&c.lag)
		if olag == 0 {
			w = 0
		}
		atomic.StoreUint64(&c.lag, uint64(float64(olag)*w+float64(lag)*(1-w)))	// 计算并存储 EWMA 值
		success := initSuccess
		if info.Err != nil && !codes.Acceptable(info.Err) {
			// 非逻辑类错误，可能是超时等，需要降低节点权重
			success = 0
		}
		osucc := atomic.LoadUint64(&c.success)
		atomic.StoreUint64(&c.success, uint64(float64(osucc)*w+float64(success)*(1-w)))	// 修正 success 的值

		// 按需打印节点日志
		stamp := p.stamp.Load()
		if now-stamp >= logInterval {
			if p.stamp.CompareAndSwap(stamp, now) {
				p.logStats()
			}
		}
	}
}
```

####	小结


##  0x03  自适应的熔断器实现


##  0x04  自适应的限流器实现


## 0x05 参考
-	[go-zero 的 P2C 算法实现](https://github.com/zeromicro/go-zero/blob/master/zrpc/internal/balancer/p2c/p2c.go)
- [自适应负载均衡算法原理与实现](https://learnku.com/articles/60059)
- [深入理解云原生下自适应限流技术原理与应用](https://www.infoq.cn/article/e6ohg7ljtttwszj0sdhi)