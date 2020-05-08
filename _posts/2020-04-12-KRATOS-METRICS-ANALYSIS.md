---
layout:     post
title:      理解 Kratos 的数据统计类型 Metrics（一）
subtitle:   分析 Kratos 框架中的 Metrics
date:       2020-04-12
author:     pandaychen
header-img:
catalog: true
category:   false
tags:
    - Metrics
---

##  0x00    前言
这篇文章，分析下 Kratos 项目中的核心统计结构 Metrics 及滑动窗口 Window 的实现。
Metrics 和 Window 的用途如下：
-	Metrics：在数据库操作指标量化（如延迟，错误，调用次数等）、接口指标量化使用较为广泛
-	Window：在限流器、熔断器、负载均衡算法判定等需要计算比率的地方使用较多，主要解决单个指标计算的平滑性，Window 的实例化结构就是 RollingXX 系列，如 RollingCounter、RollingGauge 等

##  0x01    滑动窗口的结构
和 Hystrix-Go 实现不同，Kratos 中的滑动窗口是通过单链表结构实现的。如下图所示：
![image](https://s1.ax1x.com/2020/04/14/Gzf51U.jpg)

下面看看 Window 的数据结构及封装的主要操作方法：

####	Window 中的单元节点：Bucket
Bucket 提供了 `Append` 方法，用于向 `Points` 中添加数据，`Points` 是 float64 类型的 slice，主要存放单个指标的值，如延迟，错误次数等等

```golang
type Bucket struct {
	Points []float64 // 单个节点中的统计数据（数组）
	Count  int64     // 总数
	next   *Bucket   // 链表实现
}
```

Bucket 提供了两种数值添加的接口：`Append` 和 `Add`，这两个方法会被上层 Window 的方法调用：
-	`Append` 是在 Bucket 的 `Points` 数组中直接追加数据：

```golang
// Append appends the given value to the bucket.
func (b *Bucket) Append(val float64) {
	b.Points = append(b.Points, val)
	b.Count++
}
```
-	`Add` 是在 Bucket 的 `Points` 数组中的指定 index 位置累加值：

```golang
// Add adds the given value to the point.
// 给 []float64 指定位置累加值
func (b *Bucket) Add(offset int, val float64) {
	b.Points[offset] += val
	b.Count++
}
```

Bucket 的遍历方式：
```golang
// Next returns the next bucket.
// 遍历链表用（返回当前 node 的下一个 node）
func (b *Bucket) Next() *Bucket {
	return b.next
}
```

####	Window 结构及方法
Window 的结构，就是 Bucket 组成的 slice：

```golang
type Window struct {
	window []Bucket // 滑动窗口实现
	size   int
}
```

Window 的初始化逻辑，这里有个细节是将 Bucket 初始化为环形数组（RingQueue）：
![image](https://wx1.sbimg.cn/2020/04/28/-.png)

```golang
// NewWindow creates a new Window based on WindowOpts.
// 初始化滑动窗口
func NewWindow(opts WindowOpts) *Window {
	buckets := make([]Bucket, opts.Size)
	for offset := range buckets {
		// 初始化每个 bucket
		buckets[offset] = Bucket{Points: make([]float64, 0)}
		nextOffset := offset + 1
		if nextOffset == opts.Size {
			// 构建一个 queue（环）
			nextOffset = 0
		}

		// 初始化为一个链表（首尾相接）
		buckets[offset].next = &buckets[nextOffset]
	}
	return &Window{window: buckets, size: opts.Size}
}
```

同样的，Window 的数值添加方法也只是对 Bucket 提供接口的进一步封装：
-	Window 的 `Append` 方法：向指定的偏移 offset（位于 offset 位置的 Bucket）添加值

```golang
// Append appends the given value to the bucket where index equals the given offset.
func (w *Window) Append(offset int, val float64) {
	// 调用了 Bucket 的 Append 方法
	w.window[offset].Append(val)
}
```

-	Window 的 `Add` 方法：向指定的偏移 offset 的 `0` 号位置累加值（和 `Append` 有稍许不同）

```golang
// Add adds the given value to the latest point within bucket where index equals the given offset.
func (w *Window) Add(offset int, val float64) {
	if w.window[offset].Count == 0 {
		// 如果 bucket 是空的（没有统计值），直接 Append
		w.window[offset].Append(val)
		return
	}
	// bucket 非空，在 Point[] 的 0 号位置累加值
	w.window[offset].Add(0, val)
}
```

##	0x02	window 的迭代（遍历）器实现
Window 提供了 [iterator 的封装](https://github.com/go-kratos/kratos/blob/master/pkg/stat/metric/iterator.go)，用于滑动窗口的遍历。遍历的目的是为了对窗口的数据做提取和计算；比如，计算截至当前时间滑动窗口的请求失败率，就需要遍历从窗口 start 位置到目前时间的所有 Bucket 的 $$\frac{错误总数}{请求总数}$$。

遍历器的结构如下：

```golang
// Iterator iterates the buckets within the window.
type Iterator struct {
	count         int // 遍历完成的条件（i.count != i.iteratedCount）
	iteratedCount int
	cur           *Bucket // 当前迭代器的位置
}
```

下面 `Next` 方法，定义了遍历器的退出条件：遍历完 `count` 个 Bucket（窗口）后完成，其中 `iteratedCount` 是计数器
```golang
// 这里很重要，Iter 的迭代规则是，移动 count 的次数 == 当前移动次数
// Next returns true util all of the buckets has been iterated.
func (i *Iterator) Next() bool {
	return i.count != i.iteratedCount
}
```

`Bucket` 方法，获取当前的 Bucket，并且把指针指向下一个 Bucket
```golang
// Bucket gets current bucket.
func (i *Iterator) Bucket() Bucket {
	if !(i.Next()) {
		panic(fmt.Errorf("stat/metric: iteration out of range iteratedCount: %d count: %d", i.iteratedCount, i.count))
	}
	bucket := *i.cur
	i.iteratedCount++	// 累加计数器
	i.cur = i.cur.Next()
	return bucket
}
```

##	0x03	滑动窗口 window + 时间跨度
在项目中，如何在滑动窗口中加入时间跨度，用来实现滑动窗口结构的实例化？答案就是 Rolling 结构。如下图，一个 Bucket 代表 500ms，一个滑动窗口占据 2 个 Bucket。
![image](https://s1.ax1x.com/2020/04/26/J61qD1.png)

##	0x04	滑动窗口的实例化 Rolling
[Rolling_policy](https://github.com/go-kratos/kratos/blob/master/pkg/stat/metric/rolling_policy.go) 中，封装了滑动窗口，加入了互斥锁、单位时间跨度（单个桶）、最后一次更新时间等，使其成为外部可调用的结构体，如下：
```golang
type RollingPolicy struct {
	mu     sync.RWMutex
	size   int			// 滑动窗口的 size
	window *Window		// 滑动窗口
	offset int

	bucketDuration time.Duration		// 一个桶代表多长时间
	lastAppendTime time.Time			// 滑动窗口的 START 位置
}
```

通过，`time.Now()`- 当前时间、`bucketDuration` 以及 `lastAppendTime` 这几项时间因素的关联，是的 `RollingPolicy.window` 具有了基于时间的滑动窗口的概念。

####	RollingPolicy 的方法封装

RollingPolicy 中有 `3` 个核心方法，分别为 `timespan`、`add` 和 `Reduce`。

下面 `timespan()` 方法就是计算：当前调用此方法的时刻，距离上一次写入（`lastAppendTime`）滑过了几个 Bucket
```golang
func (r *RollingPolicy) timespan() int {
	v := int(time.Since(r.lastAppendTime) / r.bucketDuration)
	if v > -1 { // maybe time backwards
		return v
	}
	// 时间调整了
	return r.size
}
```

RollingPolicy 的添加数据方法，分别调用了 window 的 `Append` 和 `Add` 方法：

```golang
// Append appends the given points to the window.
func (r *RollingPolicy) Append(val float64) {
	r.add(r.window.Append, val)
}

// Add adds the given value to the latest point within bucket.
func (r *RollingPolicy) Add(val float64) {
	r.add(r.window.Add, val)
}
```

`add` 方法：通过计算出当前时间 `time.Now()` 与 `lastAppendTime` 的跨度差，在环形滑动窗口中获取正确的位置，然后调用传入的 `f` 进行插入操作。请注意，跨度超过了 Window 的 End 位置需要从 Start 位置重新计算。

```golang
func (r *RollingPolicy) add(f func(offset int, val float64), val float64) {
	r.mu.Lock()
	// 计算时间跨度（跨过了几个 bucket）
	timespan := r.timespan()
	if timespan > 0 {
		// 当 timespan>0 时，表示有跨度
		// 更新当前 append 时间
		r.lastAppendTime = r.lastAppendTime.Add(time.Duration(timespan * int(r.bucketDuration)))
		offset := r.offset
		// reset the expired buckets
		s := offset + 1 //s 指向下一个位置
		if timespan > r.size {
			// 如果跨度超过了 window 的大小，timespan 最大为 window 的 size
			timespan = r.size
		}
		e, e1 := s+timespan, 0 // e: reset offset must start from offset+1
		if e > r.size {
			e1 = e - r.size
			e = r.size
		}
		for i := s; i < e; i++ {
			// 清理 offset1---> s+timespan 的之间的 bucket
			r.window.ResetBucket(i)
			offset = i
		}
		for i := 0; i < e1; i++ {
			// 如果超过一个跨度，那么说明时间跨度在两个区间上
			r.window.ResetBucket(i)
			offset = i
		}
		r.offset = offset
	}
	// 添加到 offset 位置
	//（当 timespan==0 时，说明，当前时间未出现 span，直接操作 r.offset 位置即可）
	f(r.offset, val)
	r.mu.Unlock()
}
```

`Reduce` 方法，非常有意思，它的传入参数是 [reduce.go](https://github.com/go-kratos/kratos/blob/master/pkg/stat/metric/reduce.go) 中定义的求值操作，如 `Sum`、`Avg`、`Min`、`Max` 和 `Couter`。
该方法的作用是，在 timespan 这个区间进行遍历，计算遍历的起始位置 `offset` 和长度 `count`，会调用上面这个几个方法之一（对遍历的这些 Bucket）进行计算，最终得到 val。比如，求当前时间 `time.Now()` 到 `lastAppendTime` 之间，滑动窗口的 Sum 累加值。
```golang
// Reduce applies the reduction function to all buckets within the window.
func (r *RollingPolicy) Reduce(f func(Iterator) float64) (val float64) {
	r.mu.RLock()
	timespan := r.timespan()
	if count := r.size - timespan; count > 0 {
		offset := r.offset + timespan + 1
		if offset >= r.size {
			offset = offset - r.size
		}
		// 计算得到遍历的开始位置 offset 和遍历长度 count
		//f 的参数，就是 Iterator 的结果
		val = f(r.window.Iterator(offset, count))
	}
	r.mu.RUnlock()
	return val
}
```

####	图解 `Reduce` 方法

1、初始化状态
![image](https://wx2.sbimg.cn/2020/05/08/reduce1.png)

2、计算需要遍历的 Bucket 个数，如图（所有深绿色的部分，滑动窗口中的旧数据，都是需要遍历累计计算值的）
$$count = r.size - r.timespan()$$

![image](https://wx1.sbimg.cn/2020/05/08/reduce2.png)

3、从 `r.offset + timespan + 1` 开始，遍历滑动窗口，遍历的个数是 `count`，这里没考虑 `offset` 超出 `r.size` 的情况（意为 `span` 的位置在 `r.offset` 之前）

![image](https://wx2.sbimg.cn/2020/05/08/reduce3.png)

4、最后，调用传入的 `Iterator` 计算值

##	0x05	总结
本文分析了 Kratos 框架中的 Metrics 基础封装、滑动窗口的实现 Window、实例化 RollingPolicy 等，下一篇文章来分析下，更高级的结构 RollingCounter、RollingGauage 等。

##  0x06    参考
-	[Kratos-Metrics](https://github.com/go-kratos/kratos/tree/master/pkg/stat/metric)

转载请注明出处，本文采用 [CC4.0](http://creativecommons.org/licenses/by-nc-nd/4.0/) 协议授权