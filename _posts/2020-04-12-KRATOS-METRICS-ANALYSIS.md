---
layout:     post
title:      理解 Kratos 的数据统计类型 Metrics（未完待续）
subtitle:   分析 Kratos 框架中的 Metrics（二）
date:       2020-04-12
author:     pandaychen
header-img:
catalog: true
category:   false
tags:
    - Metrics
---

##  0x00    前言
这边文章，分析下 Kratos 项目中的核心统计结构 Metrics 及滑动窗口 Window 的实现。
Metrics和Window的用途如下：
-	Metrics：在数据库操作指标量化（如延迟，错误，调用次数等）、接口指标量化使用较为广泛
-	Window：在限流器、熔断器、负载均衡算法判定等需要计算比率的地方使用较多，主要解决单个指标计算的平滑性

##  0x01    Kratos 中的滑动窗口实现
和 Hystrix-Go 实现不同，Kratos 中的滑动窗口是通过单链表结构实现的。如下图所示：
![image](https://s1.ax1x.com/2020/04/14/Gzf51U.jpg)

下面看看Window的数据结构及封装的主要操作方法：

>	Bucket 提供了 Append 方法，用于向 Points 中添加数据，Points是float64的slice，主要存放单个指标的值，如延迟，错误次数等等

```golang
type Bucket struct {
	Points []float64 // 单个节点中的统计数据（数组）
	Count  int64     // 总数
	next   *Bucket   // 链表实现
}

// Append appends the given value to the bucket.
func (b *Bucket) Append(val float64) {
	b.Points = append(b.Points, val)
	b.Count++
}
```

```golang
type Window struct {
	window []Bucket // 滑动窗口实现
	size   int
}
```


##	0x02	滑动窗口 window 实现
![image](https://s1.ax1x.com/2020/04/26/J61qD1.png)

##	0x03	window 的迭代器实现

```golang
// Iterator iterates the buckets within the window.
type Iterator struct {
	count         int // 遍历完成的条件（i.count != i.iteratedCount）
	iteratedCount int
	cur           *Bucket // 当前迭代器的位置
}
```
// Next returns true util all of the buckets has been iterated.

```golang
// 这里很重要，Iter 的迭代规则是，移动 count 的次数 == 当前移动次数
func (i *Iterator) Next() bool {
	return i.count != i.iteratedCount
}

// Bucket gets current bucket.
// 获取当前的 Bucket，并且把指针指向下一个 bucket
func (i *Iterator) Bucket() Bucket {
	if !(i.Next()) {
		panic(fmt.Errorf("stat/metric: iteration out of range iteratedCount: %d count: %d", i.iteratedCount, i.count))
	}
	bucket := *i.cur
	i.iteratedCount++
	i.cur = i.cur.Next()
	return bucket
}

```


##  0x04    参考
-	[Kratos-Metrics](https://github.com/go-kratos/kratos/tree/master/pkg/stat/metric)

转载请注明出处，本文采用 [CC4.0](http://creativecommons.org/licenses/by-nc-nd/4.0/) 协议授权