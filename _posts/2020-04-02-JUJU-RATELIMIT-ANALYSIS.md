---
layout:     post
title:      开源限流组件分析（三）：JuJu-Ratelimit 限速算法实现分析
subtitle:   分析一款基于令牌桶实现的限速算法
date:       2020-04-02
author:     pandaychen
header-img:
catalog: true
category:   false
tags:
    - 限流
---


##  0x00    前言
这篇文章来分析下 [juju-ratelimit](https://github.com/juju/ratelimit) 的使用及实现细节，这个 Golang 的项目基于 Token Bucket 实现了限流。核心的逻辑就两点：
1.  （初始化）生产令牌桶
2.  消费令牌的接口

![image](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/2020/0402/1-limit.png)

##	0x01	核心结构

####    Clock 结构
对 time 相关接口的封装，主要提供产生当前时间
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

####    核心：令牌桶结构 Bucket
令牌桶结构定义如下，初读此结构，会发现并没有类似 Hystrix-Go 的限速实现中的 Buffered Channel 结构，那么限速的逻辑（某个时间点桶中可用的令牌数）如何实现的？且看下面
```golang
type Bucket struct {
	clock Clock

	// startTime holds the moment when the bucket was
	// first created and ticks began.
	startTime time.Time	//bucket创建时间，仅初始化一次，用于计算相对位移

	// capacity holds the overall capacity of the bucket.
	capacity int64	//令牌桶最大容量

	// quantum holds how many tokens are added on
	// each tick.
	quantum int64	//每次放入 quantum 个令牌

	// fillInterval holds the interval between each tick.
	fillInterval time.Duration	//每次放入令牌的间隔

	// mu guards the fields below it.
	 // 互斥锁，保护下面的变量
	mu sync.Mutex

	// availableTokens holds the number of available
	// tokens as of the associated latestTick.
	// It will be negative when there are consumers
	// waiting for tokens.
	availableTokens int64   // 记录 latestTick--- 当前时间内，桶中可用的令牌数量，注意可能为负值

	// latestTick holds the latest tick for which
	// we know the number of tokens in the bucket.

	// 保存最新的 tick，也就是从创建 Bucket 到现在的时钟周期
	latestTick int64        // 核心字段：记录上一次取令牌的时间点
}
```

上面的结构中的字段，与[前文](https://pandaychen.github.io/2020/04/02/MICROSVC-RATELIMIT-INTRO/#0x03----惰性求值的意义)的公式可以对应起来看。特别注意下面几个成员：
-	`availableTokens`：可能为负值，当前可用的令牌数（为何为负值？）
-	`latestTick`：保存最新的 `tick`，也就是从创建 `Bucket` 到现在的时钟周期
-	`fillInterval`：放入令牌的间隔时间

##	0x02	对外接口

####    初始化令牌（桶）的方法
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

常用如下的初始化方法，最终都是调用`NewBucketWithQuantumAndClock`方法：
-	`NewBucket(fillInterval time.Duration, capacity int64)`：默认的初始化函数, 每一个周期内生成 `1` 个令牌, 默认 `quantum = 1`
-	`NewBucketWithQuantum(fillInterval time.Duration, capacity, quantum int64)`： 每个周期`fillInterval`内生成 `quantum` 个令牌（考虑这`quantum`个令牌是平均分布在`fillInterval`时间区间中）
-	`NewBucketWithRate(rate float64, capacity int64)`：每秒中产生 `rate` 速率的令牌，`fillInterval`默认为`1s`，`quantum`需要计算得出


```golang
func NewBucketWithQuantumAndClock(fillInterval time.Duration, capacity, quantum int64, clock Clock) *Bucket {
	// ....
	return &Bucket{
		clock:           clock,
		startTime:       clock.Now(),
		latestTick:      0,            // 上一次产生token的记录点
		fillInterval:    fillInterval, // 产生token的间隔
		capacity:        capacity,     // 桶的size
		quantum:         quantum,      // 每秒产生token的速率
		availableTokens: capacity,     // 桶内可用的令牌的个数（may <=0）
	}
}
```

这里，再看下`NewBucketWithRate`的实现，需要根据参数`rate`推算出`tb.fillInterval`以及`tb.quantum`：

| `rate` | `tb.fillInterval` | `tb.quantum` |
| :-----:| :----: | :----: |
| `10000`| `1`	| `100µs` |
| `1000`| `1`	| `1ms` |
| `100`| `1`	| `10ms` |
| `10`| `1`	| `100ms` |

```golang
// NewBucketWithRate returns a token bucket that fills the bucket
// at the rate of rate tokens per second up to the given
// maximum capacity. Because of limited clock resolution,
// at high rates, the actual rate may be up to 1% different from the
// specified rate.
func NewBucketWithRate(rate float64, capacity int64) *Bucket {
	return NewBucketWithRateAndClock(rate, capacity, nil)
}

// NewBucketWithRateAndClock is identical to NewBucketWithRate but injects a
// testable clock interface.
func NewBucketWithRateAndClock(rate float64, capacity int64, clock Clock) *Bucket {
	// Use the same bucket each time through the loop
	// to save allocations.
	tb := NewBucketWithQuantumAndClock(1, capacity, 1, clock)
	for quantum := int64(1); quantum < 1<<50; quantum = nextQuantum(quantum) {
		fillInterval := time.Duration(1e9 * float64(quantum) / rate)
		if fillInterval <= 0 {
			continue
		}
		tb.fillInterval = fillInterval
		tb.quantum = quantum
		if diff := math.Abs(tb.Rate() - rate); diff/rate <= rateMargin {
			return tb
		}
	}
	panic("cannot find suitable quantum for " + strconv.FormatFloat(rate, 'g', -1, 64))
}
```

####    消费（获取）令牌的方法
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

1.	`Available`方法：返回可用的令牌数，当有消费者在等待令牌时，将返回负值
2.	`Capacity`方法：返回令牌桶的容量
3.	`Rate`方法：返回每秒放入令牌桶的速率
4.	`Take`方法：返回取出 `count` 个令牌需要等待的时间，需要调用端进行 `sleep` 等待（注意 `Take` 方法不支持放回令牌）
5.	 `TakeAvailable`：返回可用的令牌数（不一定等于 `count`），如果没可用的令牌，将返回`0`，不会进行阻塞
6.  `TakeMaxDuration`：传入参数`maxWait`表示最大等待时间，此方法的大致逻辑如下：
	-	`TakeMaxDuration` 仅在令牌的等待时间不大于 `maxWait` 时才从存储桶中提取令牌
	-	如果令牌变得比 `maxWait` 更长的时间才可用，则不执行任何操作返回`false`
	-	否则返回调用者应等待直到令牌实际可用的时间，以及`true`
7.	`Wait`方法：阻塞等待直能获取 `count` 令牌
8.	`WaitMaxDuration`方法：`WaitMaxDuration` 仅在需要等待不超过 `maxWait` 的情况下才会从存储桶中获取令牌。返回是否已从存储桶中删除了所有令牌。如果未删除任何令牌，则立即返回。


##  0x03    代码分析

####	Rate方法
注意到，`Bucket`的成员`latestTick`、`quantum`都是以`int64`表示的，同时`time.Duration` 实际上也是以 `nanosecond` 为单位，即`1e9`。因此，`Rate`方法计算出的结果，即`1e9/float64(tb.fillInterval)` 的结果就是 `1/tb.fillInterval` 秒。<br>
于是令牌桶产生令牌的速率是: 每秒内产生 `float64(tb.quantum) / float64(tb.fillInterval)`
```golang
// Rate returns the fill rate of the bucket, in tokens per second.
func (tb *Bucket) Rate() float64 {
	return 1e9 * float64(tb.quantum) / float64(tb.fillInterval)
}
```

下面分析两种典型的消费token方法实现：

#### 	核心方法：take
`take` 是获取令牌的公共实现，其中调用了`currentTick`及`adjustavailableTokens`两个核心方法。`take`方法有三个参数：
-	`now`：当前时刻
-	`count`：获取令牌的数量
-	`maxWait`：最大等待时间（默认为最大值`0x7fffffffffffffff`）

**注意到调用`take`的前置方法都加了互斥锁，所以`take`方法是线程安全的**

1、`currentTick`方法<br>
这个是获取当前时间与坐标开始的距离，除以`tb.fillInterval`后得到这中间有多少个放置令牌的span跨度
```golang
// currentTick returns the current time tick, measured
// from tb.startTime.
func (tb *Bucket) currentTick(now time.Time) int64 {
	// 由于 tb.fillInterval 表示放入令牌的时间间隔
    // 所以这里计算的是从开始到 now 时间经历了多少个放令牌的间隔数
    // 即 tick * tb.quantum 等于当前的令牌总数
	return int64(now.Sub(tb.startTime) / tb.fillInterval)
}
```
2、`adjustavailableTokens`方法<br>
参数是`currentTick`的返回值，这个较易理解，通过`(tick - lastTick) * tb.quantum`计算在`tick`与上一次`lastTick`间，能够生产多少个令牌（从上次请求到本次请求所可能生产的令牌数）：
```golang
func (tb *Bucket) adjustavailableTokens(tick int64) {
	lastTick := tb.latestTick
	tb.latestTick = tick
	if tb.availableTokens >= tb.capacity {
		//如果令牌桶满了，则不计算了，直接返回
		return
	}

	//桶内剩余的token数量 + 新产生的token数量
	tb.availableTokens += (tick - lastTick) * tb.quantum
	// 如果产生的令牌数量超过了桶的容量，只保留桶大小数量的令牌即可
	if tb.availableTokens > tb.capacity {
		tb.availableTokens = tb.capacity
	}
	return
}
```
3、`take`方法<br>

```golang
// now 表示获取令牌的时间，也就是当前时间
// count 表示获取的令牌数量
// maxWait 表示等待的最长时间
func (tb *Bucket) take(now time.Time, count int64, maxWait time.Duration) (time.Duration, bool) {
	if count <= 0 {
		return 0, true
	}

	// 计算从 startTime 到 now 经历的 tick 时间间隔数
	tick := tb.currentTick(now)
	// 通过惰性计算，调整当前可用的令牌数；当超过令牌桶大小时，剩余的令牌数等于桶大小
	tb.adjustavailableTokens(tick)
	avail := tb.availableTokens - count
	// 当 availableTokens >= count 时，令牌可用
	if avail >= 0 {
		tb.availableTokens = avail
		return 0, true
	}
	// Round up the missing tokens to the nearest multiple
	// of quantum - the tokens won't be available until
	// that tick.

	// endTick holds the tick when all the requested tokens will
	// become available.

	// 计算下一个放入令牌的整数倍周期，防止 int64(小于 1) 为零的情况
	// 这里avail为负，表示还欠多少个令牌（本次总量为count）
	endTick := tick + (-avail+tb.quantum-1)/tb.quantum

	// 计算下一个放入令牌的时间
	endTime := tb.startTime.Add(time.Duration(endTick) * tb.fillInterval)

	// 计算需要等待的时间
	waitTime := endTime.Sub(now)

	// 当最大等待时间 < waitTime 时，直接返回false，表示无法获取到令牌数
	if waitTime > maxWait {
		return 0, false
	}
	tb.availableTokens = avail
	// 如果 waitTime != 0，调用方需要阻塞等待 waitTime 的时间，才能拿到count个令牌
	return waitTime, true
}
```

####	TakeAvailable方法
```golang

```


####	Take方法
`Take`实现了惰性求值的逻辑，注意需要加锁，获取当前时间`tb.clock.Now()`
```golang
// 返回取出 count道此一游  令牌需要等待的时间，需要调用端进行 sleep 等待，注意 Take 方法不支持放回令牌
func (tb *Bucket) Take(count int64) time.Duration {
	tb.mu.Lock()
	defer tb.mu.Unlock()
	d, _ := tb.take(tb.clock.Now(), count, infinityDuration)
	return d
}
```

####     adjustavailableTokens 方法的解释
`adjustavailableTokens`，该方法是用来计算当前时刻桶中可取的令牌总数。前面说了，令牌桶算法实现是每隔一段固定的时间向桶中放令牌，这个时间间隔足够合适，令牌的限速效果就越平滑。
这个 [方法的实现](https://github.com/juju/ratelimit/blob/master/ratelimit.go#L312)，并非使用 Buffered Channel 及其他辅助结构实现，而是采用了一种很精妙的计算方式，类似于惰性求值，计算方法描述如下：

假设令牌桶容量为 $capacity$ ，上一次放令牌的时间为 $$t_1$$（`latestTick`），当时桶中的令牌数 $$k_1$$（`availableTokens`），默认存放令牌的单位时间间隔为 $$t_i$$，每次向令牌桶中放 $$x$$ 个令牌。现在调用 `TakeAvailable()` 方法来获取令牌，将这个时刻记为 $$t_2$$。在 $$t_2$$ 时刻，令牌桶中可用的令牌数可以使用下式推算：

$$current=k_1+x*\frac{(t_2-t_1)}{t_i}$$

桶中可用的令牌数目就是 $$cuurent$$ 和 $$capacity$$ 的最大值

![image](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/2020/0402/2-limit.png)

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
		//如果超过了令牌桶容量，那么则修正
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

##  0x04  参考
-   [JUJU-limit 实现](https://github.com/juju/ratelimit)

转载请注明出处，本文采用 [CC4.0](http://creativecommons.org/licenses/by-nc-nd/4.0/) 协议授权