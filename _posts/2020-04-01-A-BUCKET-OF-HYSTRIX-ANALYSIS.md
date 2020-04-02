---
layout:     post
title:      （数据结构）Hystrix 中的 RollingCount 实现
subtitle:
date:       2020-03-31
author:     pandaychen
header-img:
catalog: true
category:   false
tags:
    - Hystrix
---


##  0x00 前言
&emsp;&emsp;Hystrix-go 中实现接口错误率计算的数据结构，是一个非常典型的 RollingCount + 滑动窗口实现的计数桶（更专业一点的称呼：**时间序列数据库**）。这篇文章来分析下其代码实现。其主要的功能 [实现代码](https://github.com/afex/hystrix-go/blob/master/hystrix/rolling/rolling.go) 在此。

##	0x01	滑动窗口

&emsp;&emsp; 默认的统计控制器 DefaultMetricCollector 保存着熔断器的所有状态，成功次数（successes），调用次数（numRequests），失败次数（failures），被拒绝次数（rejects）等等。

```golang
type DefaultMetricCollector struct {
	mutex *sync.RWMutex

	numRequests *rolling.Number
	errors      *rolling.Number

	successes               *rolling.Number		// 对应 succ 的 bucket
	failures                *rolling.Number		// 对应 failure 的 bucket
	rejects                 *rolling.Number
	shortCircuits           *rolling.Number
	timeouts                *rolling.Number
	contextCanceled         *rolling.Number
	contextDeadlineExceeded *rolling.Number

	fallbackSuccesses *rolling.Number
	fallbackFailures  *rolling.Number
	totalDuration     *rolling.Timing
	runDuration       *rolling.Timing
}
```

在上述结构定义中，所有的 `rolling.Number`（一个桶）就构造成为一个滑动窗口（多个属性）。在 Hystrix 中，一个滑动窗口，包含若干个桶（默认是 10 个），每个桶保存一定时间间隔内的统计数据（默认是 1s），如下图所示。在图中，每个矩形框代表一个桶，每个桶记录着 1 秒内的 4 个指标数据：成功量、失败量、超时量、拒绝量。这 10 个桶合起来就是一个完整的滑动窗口。
![image](https://s1.ax1x.com/2020/03/31/GQcF54.png)

实际上，每个状态，如 succ、failure、timeout 和 rejection 都各自创建了一个 bucket：
![image](https://s1.ax1x.com/2020/04/01/G15xMD.png)


##	0x02	Number 数据结构 -- 哈希桶
&emsp;&emsp; 每一种状态，都映射到自己单独的 Bucket 中，Number 的结构如下：
```golang
type Number struct {
	Buckets map[int64]*numberBucket	// 以 unixtimestamp 为 key
	Mutex   *sync.RWMutex
}

type numberBucket struct {
	Value float64
}
```
这里，每一次对熔断器的状态进行修改（更新）时，Number 都要先得到当前的 timestamp（秒级），如果 Bucket 不存在则创建。Rolling 包像个时间序列数据库，Buckets 的 Key 是 Unix 时间戳，Number 只保存 10s 内的数据。

```golang
func (r *Number) getCurrentBucket() *numberBucket {
	// 先得到当前的 timestamp
	now := time.Now().Unix()
	var bucket *numberBucket
	var ok bool
	// 判断是否存在，不存在则创建
	if bucket, ok = r.Buckets[now]; !ok {
		bucket = &numberBucket{}
		r.Buckets[now] = bucket
	}

	return bucket
}
```

此外，修改完成后去掉 10s 以外的数据
```golang
func (r *Number) removeOldBuckets() {
	now := time.Now().Unix() - 10

	for timestamp := range r.Buckets {
		// TODO: configurable rolling window
		if timestamp <= now {
			delete(r.Buckets, timestamp)
		}
	}
}
```

最后，看下，对于对每个 bucket 中的 value 值进行更新的 Increment 方法，调用了 `getCurrentBucket` 和 `removeOldBuckets` 方法，即先得到 Bucket，更新对应的计数后再删除旧的数据（Bucket）。这样就实现了，根据时间的增长，定期更新滑动窗口中的值，并且删除掉 10s 窗口之外的旧数据的功能。

此外，因为每个桶都会被多个线程并发地更新指标数据，所以桶对象需要提供一些线程安全的数据结构和更新方法。为此，原生的 Hystrix（JAVA）版本大量使用了 CAS，而 golang 版本则使用 sync.RWMutex 来控制。另外，在原生版本中，Hystrix 使用一个环形数组（Ringbuffer）来维护这些桶，类似于一个 FIFO 的队列。该数组实现有一个叫 addLast(Bucket o) 的方法，用于向环形数组的末尾追加新的桶对象，当数组中的元素个数没超过最大大小时，只是简单的维护尾指针；否则，在维护尾指针时，还要通过维护首指针，将第一个位置上元素剔除掉。而 golang 版本就使用 map 来实现，根据时间戳来模拟 FIFO：即插入时，检查当前的 timestamp，如果没有就创建，插入完成后，删除 10s 之前的 Key，这样就使用 map 模拟了 FIFO 的功能。如下面的 Increment 方法：

更新统计数据时，都是向当前最新的 bucket 中更新的，因此 hystrix 的滑动窗口类 HystrixRollingNumber 中提供了 getCurrentBucket() 方法，获取当前最新的 bucket。其执行流程和图如下：

```golang
func (r *Number) Increment(i float64) {
	if i == 0 {
		return
	}

	r.Mutex.Lock()
	defer r.Mutex.Unlock()
	// 先得到 bucket（当前 timestamp）
	b := r.getCurrentBucket()
	b.Value += i
	// 删除掉旧的
	r.removeOldBuckets()
}
```
![image](https://s1.ax1x.com/2020/04/01/G1vzyn.jpg)



##	0x03	Hystrix 计算错误率的方法
&emsp;&emsp;Hystrix-go 的配置选项提供了 ErrorPercentThreshold 参数，该参数定义为，错误百分比，请求数量大于等于 RequestVolumeThreshold 并且错误率到达这个百分比后就会启动熔断。那么这个数据是如何计算的？且跟着代码来看：

判定熔断状态，调用了 IsHealthy 方法：
```golang
if !circuit.metrics.IsHealthy(time.Now()) {
	// too many failures, open the circuit
	circuit.setOpen()
	return true
}
```

[IsHealthy 方法](https://github.com/afex/hystrix-go/blob/master/hystrix/metrics.go#L148)，判断当前系统的错误率和计算的比值：
```golang
func (m *metricExchange) IsHealthy(now time.Time) bool {
	return m.ErrorPercent(now) < getSettings(m.Name).ErrorPercentThreshold
}
```

ErrorPercent 方法，从滑动窗口中计算出错误率，定义为：（当前时间 - 滑动窗口的最左侧时间），该区间内的错误总数 / 请求总数，该值即为 ErrorPercent

```golang
func (m *metricExchange) ErrorPercent(now time.Time) int {
	m.Mutex.RLock()
	defer m.Mutex.RUnlock()

	var errPct float64
	reqs := m.requestsLocked().Sum(now)	// 传入的 now
	errs := m.DefaultCollector().Errors().Sum(now)	// 传入的 now

	if reqs > 0 {
		errPct = (float64(errs) / float64(reqs)) * 100
	}

	return int(errPct + 0.5)
}
```

而 [`Sum` 方法](https://github.com/afex/hystrix-go/blob/master/hystrix/rolling/rolling.go) 也很直观，就是遍历滑动窗口（Bucket），累加相应状态的计数：
```go
// Sum sums the values over the buckets in the last 10 seconds.
func (r *Number) Sum(now time.Time) float64 {
	sum := float64(0)

	r.Mutex.RLock()
	defer r.Mutex.RUnlock()

	for timestamp, bucket := range r.Buckets {
		// TODO: configurable rolling window --- 目前滑动窗口的范围还是不可配置的
		if timestamp >= now.Unix()-10 {
			// 确定 timestamp 在区间内
			sum += bucket.Value
		}
	}

	return sum
}
```


##	0x04	参考
-	[服务容错与保护方案 — Hystrix](https://kiswo.com/article/1030)

转载请注明出处，本文采用 [CC4.0](http://creativecommons.org/licenses/by-nc-nd/4.0/) 协议授权