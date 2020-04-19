---
layout:     post
title:      JUJU-Ratelimit 限速算法实现分析
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
这篇文章来分析下[juju-ratelimit](github.com/juju/ratelimit)的使用及实现细节，这个Golang的项目基于Token Bucket实现了限流。核心的逻辑就两点：
1.  （初始化）生产令牌桶
2.  消费令牌的接口

![image](https://s1.ax1x.com/2020/04/19/JK4QqU.jpg)

####    令牌桶结构
结构定义如下，初读此结构，会发现并没有类似Hystrix-Go的限速实现中的类似channel结构，那么限速的逻辑（某个时间点桶中可用的令牌数）如何实现的？且看下面
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
	availableTokens int64   //记录latestTick---当前时间内，桶中可用的令牌数量

	// latestTick holds the latest tick for which
	// we know the number of tokens in the bucket.
	latestTick int64        //核心字段：记录上一次取令牌的时间点
}
```

####    初始化令牌（桶）
生产令牌的接口 (https://github.com/juju/ratelimit/blob/master/ratelimit.go#L143)

```golang
func NewBucket(fillInterval time.Duration, capacity int64) *Bucket
//默认的令牌桶，fillInterval指每过多长时间向桶里放一个令牌，capacity是桶的容量，超过桶容量的部分会被直接丢弃。桶初始是满的。

func NewBucketWithQuantum(fillInterval time.Duration, capacity, quantum int64) *Bucket
//和普通的NewBucket()的区别是，每次向桶中放令牌时，是放quantum个令牌，而不是一个令牌。

func NewBucketWithRate(rate float64, capacity int64) *Bucket
//这个就有点特殊了，会按照提供的比例，每秒钟填充令牌数。例如capacity是100，而rate是0.1，那么每秒会填充10个令牌。

func NewBucketWithClock(fillInterval time.Duration, capacity int64, clock Clock) *Bucket

func NewBucketWithRateAndClock(rate float64, capacity int64, clock Clock) *Bucket

func NewBucketWithQuantumAndClock(fillInterval time.Duration, capacity, quantum int64, clock Clock) *Bucket
```

####    申请令牌

```golang
func (tb *Bucket) Take(count int64) time.Duration {}
func (tb *Bucket) TakeAvailable(count int64) int64 {}
func (tb *Bucket) TakeMaxDuration(count int64, maxWait time.Duration) (
    time.Duration, bool,
) {}
func (tb *Bucket) Wait(count int64) {}
func (tb *Bucket) WaitMaxDuration(count int64, maxWait time.Duration) bool {}
名称和功能都比较直观，这里就不再赘述了。相比于开源界更为有名的Google的Java工具库Guava中提供的ratelimiter，这个库不支持令牌桶预热，且无法修改初始的令牌容量，所以可能个别极端情况下的需求无法满足。但在明白令牌桶的基本原理之后，如果没办法满足需求，相信你也可以很快对其进行修改并支持自己的业务场景。

```


##  0x01    代码分析

####     adjustavailableTokens：计算当前时刻桶中可取的令牌总数
前面说了，令牌桶算法实现是每隔一段固定的时间向桶中放令牌，这个时间间隔足够合适，令牌的限速效果就越平滑。
这个[方法的实现](https://github.com/juju/ratelimit/blob/master/ratelimit.go#L312)，并非使用定时器，而是采用了一种很精妙的计算方式，类似于惰性求值，计算方法描述如下：

假设令牌桶容量为$$capacity$$，上一次放令牌的时间为 $$t_1$$（latestTick），当时桶中的令牌数$$k_1$$（availableTokens），默认存放令牌的单位时间间隔为$$t_i$$，每次向令牌桶中放$$x$$个令牌。现在调用`TakeAvailable()`方法来获取令牌，将这个时刻记为$$t_2$$。在$$t_2$$时刻，令牌桶中可用的令牌数可以使用下式推算：
$$current=k_1+x*\frac{(t_2-t_1)}{t_i}$$

桶中可用的令牌数目就是cuurent 和 capacity的最大值

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
    //采用惰性求值的方法
	tb.availableTokens += (tick - lastTick) * tb.quantum
	if tb.availableTokens > tb.capacity {
		tb.availableTokens = tb.capacity
	}
	return
}
```

##  0x02  参考
-   [JUJU-limit实现](https://github.com/juju/ratelimit)