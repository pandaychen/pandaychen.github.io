---
layout:     post
title:      开源限流组件分析（二）：Golang-time/rate 限速算法实现分析
subtitle:   分析 Golang 标准库提供的令牌桶限流器
date:       2020-04-05
author:     pandaychen
header-img: img/super-mario.jpg
catalog: true
category:   false
tags:
    - 限流
    - Golang
---

##  0x00    前言
这篇文章来分析下标准库 [time/rate](https://github.com/golang/time/blob/master/rate/rate.go) 的使用及实现细节，此库同样基于令牌桶（Token Bucket）实现了限流。

##	0x01	time/rate 的使用

####	创建限流器
使用 `NewLimiter(r Limit, b int)` 创建限速器，令牌桶容量为 `b`。初始化状态下桶是满的，即桶里装有 `b` 个令牌，以后再以每秒往里面填充 `r` 个令牌。有两种特例：
1.	允许声明容量为 0 的限速器，此时将以拒绝所有事件操作
2.	就是 `r == Inf` 时，此时 b 参数将被忽略（`const Inf = Limit(math.MaxFloat64)`），即令牌桶无限大

####	限流判定
time/rate 库提供了三类方法（其中 `AllowN`、`ReserveN` 和 `WaitN` 允许消费 n 个令牌）：
-	`Wait/WaitN`：当没有可用或足够的 Token 时，将阻塞等待 Token 或者超时取消（推荐实际程序中使用这个方法）
-	`Allow/AllowN`：当没有可用或足够的 Token 时，返回 `false`
-	`Reserve/ReserveN` 当没有可用或足够的 Token 时，返回 `Reservation` 对象，和要等待多久才能获得足够的 Token（给用户的控制权是最多的）

注意 `Wait` 方法中的阻塞等待，因为令牌桶的实现是基于时间戳的（等的越久 Token 越多），`Wait` 会返回阻塞等待的时间跨度，在此之后就可以拿到足够的令牌，配和 `context.Context` 使用效果极好。

####	使用例子
以上使用例子 [见此]()

##	0x02	令牌桶的本质
从上一篇文章 [JuJu-Ratelimit 限速算法实现分析](https://pandaychen.github.io/2020/04/02/JUJU-RATELIMIT-ANALYSIS/) 的总结，令牌桶的实现本质就是利用了 <font color="#dd0000">Token 数可以和时间跨度相互转化 </font> 的原理。需要有如下关键信息：

-	生产 Token 令牌的速率：一秒钟可以产生多少 Token（生产一个 Token 需要多长时间单位），记为 $p$（`1s`）
-	Token 令牌桶的大小 $Bucket_{size}$

基于上面这两个基础信息，容易得到：
1. 生成 $N$ 个新的 Token 一共需要的时间单位：$\frac{N}{p}*1s$
2. 给定一段时长 $Duration$，这段时间一共可以生成多少个 Token，$\frac{Duration}{1s}*p$

##	0x03	分析
同 juju-ratelimit 的实现一样，在 `timer/rate` 实现中, 并没有单独维护一个 Timer，而是采用了 lazyload 的方式，直到每次消费之前才根据时间差计算并更新 Token 数目，而且也不是用 BlockingQueue 来存放 Token，而是仅仅通过计数的方式来实现时间与令牌的换算。

注意上面这句话：<font color="#dd0000"> 直到每次消费之时根据时间差（本次减去上次保存的时间戳位置）计算 Token 数目 </font>，所以令牌桶的实现是严格依赖于时间的准确性的。

####	基础结构
`Limit` 为令牌桶的定义，在单机限流实践中通常定义为一个全局对象：
```golang
type Limit float64

type Limiter struct {
	// 初始化 NewLimiter 传入的两个值：limit 和 burst
	limit Limit	// 每秒中生产 token 的个数
	burst int // 桶的总大小

	mu     sync.Mutex
	tokens float64 // 桶中目前剩余的 token 数目，可以为负数（负数表示有些原子请求了数目过大的令牌）
	// last is the last time the limiter's tokens field was updated
	last time.Time
	// lastEvent is the latest time of a rate-limited event (past or future)
	lastEvent time.Time
}
```

####	基础接口
1、`durationFromTokens` 方法，计算生成 `N` 个新的 Token 一共需要的时间，即上一节的 $\frac{N}{p}*1s$，注意这里转为 `time.Nanosecond` 为单位了，颗粒度极小。而 `limit` 本身就是 `float64` 类型 <br>
```golang
// durationFromTokens is a unit conversion function from the number of tokens to the duration
// of time it takes to accumulate them at a rate of limit tokens per second.
// 将 token 转化为所需等待时间
func (limit Limit) durationFromTokens(tokens float64) time.Duration {
	seconds := tokens / float64(limit)
	return time.Nanosecond * time.Duration(1e9*seconds)
}
```

2、`tokensFromDuration` 方法，用于计算给定一段时长 `time.Duration`，这段时长内一共可以生成多少个令牌 Token，即上一节的 $\frac{Duration}{1s}*p$。注意这里的除法可能导致的 [精度丢失问题](https://www.cyhone.com/articles/analisys-of-golang-rate/)：<br>
```golang
// tokensFromDuration is a unit conversion function from a time duration to the number of tokens
// which could be accumulated during that duration at a rate of limit tokens per second.
func (limit Limit) tokensFromDuration(d time.Duration) float64 {
	// Split the integer and fractional parts ourself to minimize rounding errors.
	// See golang.org/issues/34861.
	// 如果是用 d.Seconds() * float64(limit), 因为 d.Seconds 是 float64 的。因此会造成精度的损失。
	// time.Duration 是 int64 类型的，表示纳秒
	// time.Second
	sec := float64(d/time.Second) * float64(limit)
	nsec := float64(d%time.Second) * float64(limit)
	return sec + nsec/1e9
}
```

3、`Every` 方法，提供时间对令牌的转换接口
```golang
// Every converts a minimum time interval between events to a Limit.
// 可以将时间转化为速率
// 例如：每 5 秒一个，转化为速率就是 0.2 一秒
func Every(interval time.Duration) Limit {
	if interval <= 0 {
		return Inf
	}
	return 1 / Limit(interval.Seconds())
}
```

3、`advance` 方法：传入参数 `now` 为当前时间，该方法是获取到 `now` 为止，可用的 Token 的令牌个数（根据上面两个基础方法计算得到）：
需要的关键参数：
-	令牌桶结构中保存了 <font color="#dd0000"> 上一次原子操作成功获取令牌 </font> 操作的时间 `lim.last`
-	传入参数为当前时间 `now time.Time`

上面两个时间跨度数据相减，就拿到了 <font color="#dd0000"> 两个相邻的原子操作之间，一共产生的令牌数目，再和令牌桶的 size 做比较（取最小值），就得到了最终可用的 Token 个数 </font>。

```golang
// advance calculates and returns an updated state for lim resulting from the passage of time.
// lim is not changed.
// @param now
// @return newNow 似乎还是这个 now，没变
// @return newLast 如果 last > now, 则 last 为 now
// @return newTokens 当前桶中应有的数目
func (lim *Limiter) advance(now time.Time) (newNow time.Time, newLast time.Time, newTokens float64) {
	// last 代表上一个取的时候的时间
	last := lim.last
	if now.Before(last) {
		last = now
	}

	// Avoid making delta overflow below when last is very old.
	// maxElapsed 表示：将 Token 桶填满需要多久
	// 为什么要拆分两步做，是为了防止后面的 delta 溢出
	// 因为默认情况下，last 为 0，此时 delta 算出来的，会非常大
	maxElapsed := lim.limit.durationFromTokens(float64(lim.burst) - lim.tokens)

	// elapsed 表示从当前到上次一共过去了多久
	// 当然了，elapsed 不能大于将桶填满的时间
	elapsed := now.Sub(last)
	if elapsed > maxElapsed {
		elapsed = maxElapsed
	}

	// Calculate the new number of tokens, due to time that passed.
	// 计算下过去这段时间，一共产生了多少 token
	delta := lim.limit.tokensFromDuration(elapsed)

	// token 取 burst 最大值，因为显然 token 数不能大于桶容量
	tokens := lim.tokens + delta
	if burst := float64(lim.burst); tokens > burst {
		tokens = burst
	}

	return now, last, tokens
}
```

##	0x04	reserveN 方法
Token 的消费方式有如下 3 种。在内部实现，最终都调用了 `reserveN` 函数来生成和消费 Token：
-	`Wait/WaitN`
-	`Allow/AllowN`
-	`Reserve/ReserveN`

```golang
// reserveN is a helper method for AllowN, ReserveN, and WaitN.
// maxFutureReserve specifies the maximum reservation wait duration allowed.
// reserveN returns Reservation, not *Reservation, to avoid allocation in AllowN and WaitN.
//
// @param now 当前消费的时间
// @param n 要消费的 token 数量
// @param maxFutureReserve 愿意等待的最长时间
func (lim *Limiter) reserveN(now time.Time, n int, maxFutureReserve time.Duration) Reservation {
	lim.mu.Lock()

	// 如果没有限制
	if lim.limit == Inf {
		lim.mu.Unlock()
		return Reservation{
			ok:        true,
			lim:       lim,
			tokens:    n,
			timeToAct: now,
		}
	}

	// 通过 advance 拿到 now 到 lim.last 之前跨度一共可用的 token 数（<= 令牌桶个数）
	now, last, tokens := lim.advance(now)

	// Calculate the remaining number of tokens resulting from the request.
	// 看下取完之后，桶还能剩能下多少 token
	tokens -= float64(n)

	// Calculate the wait duration
	// 如果 token < 0, 说明目前的 token 不够，需要等待一段时间
	var waitDuration time.Duration
	if tokens < 0 {
		// durationFromTokens：将 tokens 转为时间
		waitDuration = lim.limit.durationFromTokens(-tokens)
	}

	// Decide result
	// n<=lim.burst ：申请的 token 是否超过了桶的大小
	// waitDuration <= maxFutureReserve：需要等待的时间是否小于用户期望的时间
	ok := n <= lim.burst && waitDuration <= maxFutureReserve

	// Prepare reservation
	r := Reservation{
		ok:    ok,
		lim:   lim,
		limit: lim.limit,
	}

	// timeToAct 表示当桶中满足 token 数目等于 n 的时间
	if ok {
		r.tokens = n	//n（用户传入）
		r.timeToAct = now.Add(waitDuration)
		//r.timeToAct 表示用于需要等待到这个时刻才能获得期望大小的 token 数目（当然 waitDuration 有可能为 0，就是立即满足）
	}

	// Update state
	// 更新桶里面的 token 数目
	if ok {
		lim.last = now
		lim.tokens = tokens
		lim.lastEvent = r.timeToAct
	} else {
		// 不满足，只更新 last，last 的规则在 advance 方法中
		lim.last = last
	}

	lim.mu.Unlock()
	// 将 Reservation 对象返回
	return r
}
```


##	0x05	（消费）接口分析
基于 `reserveN` 的实现，本小节看下对外接口对其的调用方式：
####	Allow 系列
`Allow` 和 `AllowN` 的实现，最终调用了 `reserveN(now, 1, 0).ok` 和 `reserveN(now, n, 0).ok`，这两个方法只需要 `ok` 这个结果（即 Token 令牌拿到与否），比较直观：
```golang
// Allow is shorthand for AllowN(time.Now(), 1).
func (lim *Limiter) Allow() bool {
	// 传入 time.Now()
	return lim.AllowN(time.Now(), 1)
}

// AllowN reports whether n events may happen at time now.
// Use this method if you intend to drop / skip events that exceed the rate limit.
// Otherwise use Reserve or Wait.
func (lim *Limiter) AllowN(now time.Time, n int) bool {
	return lim.reserveN(now, n, 0).ok
}
```

####	Wait 系列


####	Reserve 系列

##	0x06	Reservation 结构
```golang
// A Reservation holds information about events that are permitted by a Limiter to happen after a delay.
// A Reservation may be canceled, which may enable the Limiter to permit additional events.
type Reservation struct {
	ok        bool
	lim       *Limiter
	tokens    int
	timeToAct time.Time
	// This is the Limit at reservation time, it can change later.
	limit Limit
}

// OK returns whether the limiter can provide the requested number of tokens
// within the maximum wait time.  If OK is false, Delay returns InfDuration, and
// Cancel does nothing.
func (r *Reservation) OK() bool {
	return r.ok
}

// Delay is shorthand for DelayFrom(time.Now()).
func (r *Reservation) Delay() time.Duration {
	return r.DelayFrom(time.Now())
}

// InfDuration is the duration returned by Delay when a Reservation is not OK.
const InfDuration = time.Duration(1<<63 - 1)

// DelayFrom returns the duration for which the reservation holder must wait
// before taking the reserved action.  Zero duration means act immediately.
// InfDuration means the limiter cannot grant the tokens requested in this
// Reservation within the maximum wait time.
func (r *Reservation) DelayFrom(now time.Time) time.Duration {
	if !r.ok {
		return InfDuration
	}
	delay := r.timeToAct.Sub(now)
	if delay < 0 {
		return 0
	}
	return delay
}

// Cancel is shorthand for CancelAt(time.Now()).
func (r *Reservation) Cancel() {
	r.CancelAt(time.Now())
	return
}

// CancelAt indicates that the reservation holder will not perform the reserved action
// and reverses the effects of this Reservation on the rate limit as much as possible,
// considering that other reservations may have already been made.
func (r *Reservation) CancelAt(now time.Time) {
	if !r.ok {
		return
	}

	r.lim.mu.Lock()
	defer r.lim.mu.Unlock()

	if r.lim.limit == Inf || r.tokens == 0 || r.timeToAct.Before(now) {
		return
	}

	// calculate tokens to restore
	// The duration between lim.lastEvent and r.timeToAct tells us how many tokens were reserved
	// after r was obtained. These tokens should not be restored.
	// 为什么新分配的就不算呢？
	// 因为可以 cancel 表示该 Event 尚未发生，如果已经发生，则在前面的 if 分支就 return 了;
	// 那么后面继续申请的 Event.timeToAct 必定大于当前的 r.timeToAct，也是预支的;
	// 那么归还当前的 token 时，需要把已经预支的那部分除去，因为已经算是预消费了，不能再给后面申请的 Event 使用
	restoreTokens := float64(r.tokens) - r.limit.tokensFromDuration(r.lim.lastEvent.Sub(r.timeToAct))

	// 当小于 0，表示已经都预支完了，不能归还了
	if restoreTokens <= 0 {
		return
	}
	// advance time to now
	now, _, tokens := r.lim.advance(now)
	// calculate new number of tokens
	tokens += restoreTokens
	if burst := float64(r.lim.burst); tokens > burst {
		tokens = burst
	}

	// update state
	r.lim.last = now // 这一点也很关键
	r.lim.tokens = tokens

	// 如果都相等，说明跟没消费一样。直接还原成上次的状态吧
	if r.timeToAct == r.lim.lastEvent {
		prevEvent := r.timeToAct.Add(r.limit.durationFromTokens(float64(-r.tokens)))
		if !prevEvent.Before(now) {
			r.lim.lastEvent = prevEvent
		}
	}

	return
}
```

##  0x07  参考
-	[time/rate 源码](https://github.com/golang/time/blob/master/rate/rate.go)
-	[Golang 标准库限流器 time/rate 实现剖析](https://www.cyhone.com/articles/analisys-of-golang-rate/)
-	[系统库golang.org/x/time/rate 限频器bug](https://cloud.tencent.com/developer/article/1890999)


转载请注明出处，本文采用 [CC4.0](http://creativecommons.org/licenses/by-nc-nd/4.0/) 协议授权