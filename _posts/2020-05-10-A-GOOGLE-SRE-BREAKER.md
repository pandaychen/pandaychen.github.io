---
layout:     post
title:      Google SRE 弹性熔断算法实现分析（未完待续）
subtitle:
date:       2020-05-10
author:     pandaychen
header-img: img/golang-tools-fun.png
catalog: true
category:   false
tags:
    - Ratelimit
---

##  0x00    前言
&emsp;&emsp; 这篇文章，了解下 Google SRE 中的过载保护（弹性熔断）的处理机制 --[Handling Overload](https://landing.google.com/sre/sre-book/chapters/handling-overload)

之前的文章，分析了 Hystrix-Go 熔断算法的实现，其算法核心是：当请求失败比率达到一定阈值之后，熔断器开启，并休眠一段时间（由配置决定），这段休眠期过后，熔断器将处于半开状态，在此状态下将试探性的放过一部分流量，如果这部分流量调用成功后，再次将熔断器关闭，否则熔断器继续保持开启并进入下一轮休眠周期。

但这个熔断算法有一个问题，过于一刀切。是否可以做到在熔断器开启状态下（但是后端未 Shutdown）仍然可以放行少部分流量呢？当然，这里有个前提，需要看后端此时还能够接受多少流量。下一步我们来看看 Google 的实现。

##  0x01    Google 的做法

We implemented client-side throttling through a technique we call adaptive throttling：该算法称为（客户端）自适应限流。

当客户端检测到最近的请求出现错误都是 "out of quota（配额不足）"，说明可能这个客户端超过了资源配额，后端任务会快速拒绝请求，返回 “配额不足” 的错误，有可能后端忙着不
停发送拒绝请求，导致过载；
• 依赖的资源出现大量错误，处于对下游的保护

客户端自行限制请求速度，限制生成请求的数量, 超过这个数量的请求直接在本地回复失败，而不会真是发送到服务端

该算法统计的指标依赖如下两种，每个客户端记录过去两分钟内的以下信息（一般代码中以滑动窗口实现）：
-   requests（客户端请求总量）
The number of requests attempted by the application layer(at the client, on top of the adaptive throttling system)

-   accepts（成功的请求总量 - 被 accepted）
The number of requests accepted by the backend

该算法的通用描述如下：
-	在通常情况下 $$requests==accepts$$
-	当后端出现异常情况时，$$accepts$$ 的数量会逐渐小于 $$requests$$
-	当后端持续异常时，客户端可以继续发送请求直到 $$requests=K*accepts$$，一旦超过这个值，客户端就启动自适应限流机制，新产生的请求在本地会以 $$p$$ 概率（上面的 Client request rejection probability）被拒绝
-	当客户端主动丢弃请求时，$$requests$$ 值会一直增大，在某个时间点会超过 $$K*accepts$$，使概率 $$p$$ 计算出来的值大于 0，此时客户端会以此概率对请求做主动丢弃
-	当后端逐渐恢复时，$$accepts$$ 增加，（同时 $$requests$$ 值也会增加，但是由于 $$K$$ 的关系，$$K*accepts$$ 的放大倍数更快），使得 $$\frac{requests-K*accepts}{requests+1}$$ 变为负数，从而概率 $$p==0$$，客户端自适应限流结束

客户端请求拒绝的概率（Client request rejection probability），基于如下公式计算（其中 $$K$$ 为倍率 - multiplier，常用的值为 `2`）：
$$max(0,\frac{requests-K*accepts}{requests+1})$$
该公式的解释如下：
当 $$requests-K*accepts>=0$$ 时，概率 $$p==0$$，客户端不会主动丢弃请求；反之，则概率 $$p$$，会随着 $$accepts$$ 值的变小而增加，即成功接受的请求数越少，本地丢弃请求的概率就越高。

从 Google 的文档描述中，该算法在实际中使用效果极为良好，可以使整体上保持一个非常稳定的请求速率。对于后端而言，调整 $$K$$ 值可以使得自适应限流算法适配不同的后端。关于 $$K$$ 值的意义：
-   Reducing the multiplier will make adaptive throttling behave more aggressively
-   Increasing the multiplier will make adaptive throttling behave less aggressively

翻译上面两句话就是：
-	降低 $$K$$ 值会使自适应限流算法更加激进（允许客户端在算法启动时拒绝更多本地请求）
-	增加 $$K$$ 值会使自适应限流算法不再那么激进（允许服务端在算法启动时尝试接收更多的请求，与上面相反）


##  0x02    代码实现

```golang
func (b *sreBreaker) Allow() error {
	success, total := b.summary()
	k := b.k * float64(success)
	if log.V(5) {
		log.Info("breaker: request: %d, succee: %d, fail: %d", total, success, total-success)
	}
	// check overflow requests = K * success
	if total <b.request || float64(total) < k {
		if atomic.LoadInt32(&b.state) == StateOpen {
			atomic.CompareAndSwapInt32(&b.state, StateOpen, StateClosed)
		}
		return nil
	}
	if atomic.LoadInt32(&b.state) == StateClosed {
		atomic.CompareAndSwapInt32(&b.state, StateClosed, StateOpen)
	}
	dr := math.Max(0, (float64(total)-k)/float64(total+1))
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

##  0x03    效果展示 && 总结

##  参考
-   [Handling Overload](https://landing.google.com/sre/sre-book/chapters/handling-overload/#eq2101)