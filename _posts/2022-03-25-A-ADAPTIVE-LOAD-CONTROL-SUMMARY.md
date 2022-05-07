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
自适应的负载均衡，意为自动的选择指标最优的节点进行请求（如负载低、延时低等），负载低考虑 CPU / 内存，延时主要指接口响应；此外，要求可以动态发现后端节点以及隔离故障节点。从代码 [实现]() 看，go-zero 采用了 P2C 算法 + EWMA 来实现自适应的 LB 机制。

####  基础
- P2C：在多个节点中随机选择 2 个，然后再此 2 中选择一个最优
- EWMA： 指数移动加权平均法，其意义在于只需要保存最近一次的数值（如接口延时），利用加权因子来预估时间区间的平均值，算法的具体含义可参考 [此文](https://www.cnblogs.com/jiangxinyang/p/9705198.html)。EWMA 是指各数值的加权系数随时间呈指数递减，** 越靠近当前时刻的数值加权系数就越大 **，体现了最近一段时间内的平均值。

####  EWMA 的意义
为啥要是用 EWMA 呢？试想一下，当客户端发起请求时，实际上只能通过历史的数据（如上 `N` 次的延迟，上 `N` 次的服务端负载数据）来 "预测" 本次请求的延迟，所以 EWNA 算法是比较契合的。

1、公式<br>

$$
V_{t} = \beta  \times V_{t-1} + (1- \beta) \times \theta_{t}
$$

####  β 定义
go-zero 采用的是 ** 牛顿冷却定律中的衰减函数模型计算 ** EWMA 算法中的 β 值（ Δt 为两次请求的间隔，e，k 为常数），对应的代码为：


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

  //pick c1
	atomic.StoreInt64(&c1.pick, start)
	return c1
}
```

##  0x03  自适应的熔断器实现


##  0x04  自适应的限流器实现


## 0x05 参考
-	[go-zero 的 P2C 算法实现](https://github.com/zeromicro/go-zero/blob/master/zrpc/internal/balancer/p2c/p2c.go)
- [自适应负载均衡算法原理与实现](https://learnku.com/articles/60059)
- [深入理解云原生下自适应限流技术原理与应用](https://www.infoq.cn/article/e6ohg7ljtttwszj0sdhi)