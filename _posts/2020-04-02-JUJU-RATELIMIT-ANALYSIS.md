---
layout:     post
title:      JuJu-Ratelimit 限速算法实现分析
subtitle:   分析一款基于令牌桶实现的限速算法
date:       2020-04-02
author:     pandaychen
header-img:
catalog: true
category:   false
tags:
    - Ratelimit
---

##  0x00    前言
这篇文章来分析下 [juju-ratelimit](https://github.com/juju/ratelimit) 的使用及实现细节，这个 Golang 的项目基于 Token Bucket 实现了限流。核心的逻辑就两点：
1.  （初始化）生产令牌桶
2.  消费令牌的接口

![image](https://s1.ax1x.com/2020/04/19/JKIGg1.jpg)

####    Clock 结构
对 time 相关接口的封装
```golang
// Clock represents the passage of time in a way that
// can be faked out for tests.
type Clock interface {
	// Now returns the current time.
	Now() time.Time
	// Sleep sleeps for at least the given duration.
	Sleep(d time.Duration)
}

// realClock implements Clock in terms of standard time functions
// 经典 struct{} 和 interface{} 搭配
type realClock struct{}

// Now implements Clock.Now by calling time.Now.
func (realClock) Now() time.Time {
	return time.Now()
}

// Now implements Clock.Sleep by calling time.Sleep.
func (realClock) Sleep(d time.Duration) {
	time.Sleep(d)
}
```

####    令牌桶结构
令牌桶结构定义如下，初读此结构，会发现并没有类似 Hystrix-Go 的限速实现中的 Buffered Channel 结构，那么限速的逻辑（某个时间点桶中可用的令牌数）如何实现的？且看下面
```golang
type Bucket struct {
	clock Clock

	// startTime holds the moment when the bucket was
	// first created and ticks began.
	startTime time.Time

	// capacity holds the overall capacity of the bucket.
	capacity int64

	// quantum holds how many tokens are added on
	// each tick.
	quantum int64

	// fillInterval holds the interval between each tick.
	fillInterval time.Duration

	// mu guards the fields below it.
	mu sync.Mutex

	// availableTokens holds the number of available
	// tokens as of the associated latestTick.
	// It will be negative when there are consumers
	// waiting for tokens.
	availableTokens int64   // 记录 latestTick--- 当前时间内，桶中可用的令牌数量

	// latestTick holds the latest tick for which
	// we know the number of tokens in the bucket.
	latestTick int64        // 核心字段：记录上一次取令牌的时间点
}
```

####    初始化令牌（桶）
生产令牌的 [接口](https://github.com/juju/ratelimit/blob/master/ratelimit.go#L143)

```golang
// 默认，fillInterval 指每过多长时间向桶里放一个令牌，capacity 是桶的容量，超过桶容量的部分会被直接丢弃。桶初始是满的
func NewBucket(fillInterval time.Duration, capacity int64) *Bucket

// 和普通的 NewBucket 的区别是，每次向桶中放令牌时，是放 quantum 个令牌，而不是一个令牌。
func NewBucketWithQuantum(fillInterval time.Duration, capacity, quantum int64) *Bucket

// 会按照提供的比例，每秒钟填充令牌数。例如 capacity 是 100，而 rate 是 0.1，那么每秒会填充 10 个令牌。
func NewBucketWithRate(rate float64, capacity int64) *Bucket

func NewBucketWithClock(fillInterval time.Duration, capacity int64, clock Clock) *Bucket

func NewBucketWithRateAndClock(rate float64, capacity int64, clock Clock) *Bucket

func NewBucketWithQuantumAndClock(fillInterval time.Duration, capacity, quantum int64, clock Clock) *Bucket
```

####    消费令牌
下面几个方法，底层都是调用了 [`take()` 方法](https://github.com/juju/ratelimit/blob/master/ratelimit.go#L275)，在下面 part 中分析：
```golang
// 取令牌的阻塞方法
func (tb *Bucket) Take(count int64) time.Duration {}

// TakeAvailable takes up to count immediately available tokens from the
// bucket. It returns the number of tokens removed, or zero if there are
// no available tokens. It does not block.
// 非阻塞的取令牌
func (tb *Bucket) TakeAvailable(count int64) int64 {}

func (tb *Bucket) TakeMaxDuration(count int64, maxWait time.Duration) (
    time.Duration, bool,
) {}

func (tb *Bucket) Wait(count int64) {}

func (tb *Bucket) WaitMaxDuration(count int64, maxWait time.Duration) bool {}
```


##  0x01    代码分析

####     adjustavailableTokens 方法
`adjustavailableTokens`，该方法是用来计算当前时刻桶中可取的令牌总数。前面说了，令牌桶算法实现是每隔一段固定的时间向桶中放令牌，这个时间间隔足够合适，令牌的限速效果就越平滑。
这个 [方法的实现](https://github.com/juju/ratelimit/blob/master/ratelimit.go#L312)，并非使用 Buffered Channel 及其他辅助结构实现，而是采用了一种很精妙的计算方式，类似于惰性求值，计算方法描述如下：

假设令牌桶容量为 $$capacity$$，上一次放令牌的时间为 $$t_1$$（`latestTick`），当时桶中的令牌数 $$k_1$$（`availableTokens`），默认存放令牌的单位时间间隔为 $$t_i$$，每次向令牌桶中放 $$x$$ 个令牌。现在调用 `TakeAvailable()` 方法来获取令牌，将这个时刻记为 $$t_2$$。在 $$t_2$$ 时刻，令牌桶中可用的令牌数可以使用下式推算：

$$current=k_1+x*\frac{(t_2-t_1)}{t_i}$$

桶中可用的令牌数目就是 $$cuurent$$ 和 $$capacity$$ 的最大值

![image](https://s1.ax1x.com/2020/04/19/JKVgj1.png)

```golang
// adjustavailableTokens adjusts the current number of tokens
// available in the bucket at the given time, which must
// be in the future (positive) with respect to tb.latestTick.
func (tb *Bucket) adjustavailableTokens(tick int64) {
	lastTick := tb.latestTick
	tb.latestTick = tick
	if tb.availableTokens >= tb.capacity {
		return
    }
    // 采用惰性求值的方法
	tb.availableTokens += (tick - lastTick) * tb.quantum
	if tb.availableTokens > tb.capacity {
		//availableTokens 保存当前时间 tick 令牌桶中可用的数目
		tb.availableTokens = tb.capacity
	}
	return
}
```

再看下 `adjustavailableTokens` 中的参数 tick，是如何计算的，我们从入口函数 `TakeAvailable` 来看，它的参数是消费的令牌个数：
```golang
// TakeAvailable takes up to count immediately available tokens from the
// bucket. It returns the number of tokens removed, or zero if there are
// no available tokens. It does not block.
func (tb *Bucket) TakeAvailable(count int64) int64 {
	// lock
	tb.mu.Lock()
	defer tb.mu.Unlock()
	//tb.clock.Now() 就是 time.Now()
	return tb.takeAvailable(tb.clock.Now(), count)
}

// takeAvailable is the internal version of TakeAvailable - it takes the
// current time as an argument to enable easy testing.
func (tb *Bucket) takeAvailable(now time.Time, count int64) int64 {
	if count <= 0 {
		return 0
	}
	tb.adjustavailableTokens(tb.currentTick(now))
	if tb.availableTokens <= 0 {
		return 0
	}
	if count > tb.availableTokens {
		count = tb.availableTokens
	}
	tb.availableTokens -= count
	return count
}
```

`currentTick` 方法是计算 $$\frac{t_2-t_1}{t_i}$$ 的值

```golang
// currentTick returns the current time tick, measured from tb.startTime
func (tb *Bucket) currentTick(now time.Time) int64 {
	return int64(now.Sub(tb.startTime) / tb.fillInterval)
}
```

####	take 方法
提个问题，`waitTime`这个值做何用？
```golang
// take is the internal version of Take - it takes the current time as
// an argument to enable easy testing.
func (tb *Bucket) take(now time.Time, count int64, maxWait time.Duration) (time.Duration, bool) {
	if count <= 0 {
		return 0, true
	}

	tick := tb.currentTick(now)
	tb.adjustavailableTokens(tick)
	avail := tb.availableTokens - count
	if avail >= 0 {
		tb.availableTokens = avail
		return 0, true
	}
	// Round up the missing tokens to the nearest multiple
	// of quantum - the tokens won't be available until
	// that tick.

	// endTick holds the tick when all the requested tokens will
	// become available.
	endTick := tick + (-avail+tb.quantum-1)/tb.quantum
	endTime := tb.startTime.Add(time.Duration(endTick) * tb.fillInterval)
	waitTime := endTime.Sub(now)
	if waitTime > maxWait {
		return 0, false
	}

	//更新桶中可用令牌数
	tb.availableTokens = avail
	return waitTime, true
}
```

##  0x02  参考
-   [JUJU-limit 实现](https://github.com/juju/ratelimit)

转载请注明出处，本文采用 [CC4.0](http://creativecommons.org/licenses/by-nc-nd/4.0/) 协议授权