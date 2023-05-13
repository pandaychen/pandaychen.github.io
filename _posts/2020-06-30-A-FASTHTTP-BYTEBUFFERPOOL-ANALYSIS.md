---
layout:     post
title:      Fasthttp 高性能 HttpServer 最佳实践之二：bytebufferpool
subtitle:   分析 Fasthttp 的 byte 对象池实现
date:       2020-06-30
author:     pandaychen
header-img: img/golang-horse-fly.png
catalog: true
category:   false
tags:
    - Fasthttp
    - Pool
    - 性能优化
    - bytebufferpool
---


##  0x00    前言
考虑这个普遍场景：在 Golang 开发中，在从 `io.Reader` 中读取数据时，通常要创建一个字节切片 `[]byte` 去存储，在高频调用或并发比较高的场景中，需要频繁的进行内存申请和释放（频繁的创建`[]byte`），增大了 GC 的压力；此时需要采用字节池来优化

```GO
func main() {
    r, _ := os.Open("some_bigfile")
    buf := make([]byte, 64)
    for {
        n, err := r.Read(buf)
        if err != nil {
            if err == io.EOF {
                break
            }
            log.Fatal(err)
        }
    }
}
```

前文[Golang 中的 sync.Pool 使用与实现分析](https://pandaychen.github.io/2020/03/11/GOLANG-SYNC-POOL-USAGE/)讨论了标准库`sync.Pool`的使用，`sync.Pool` 是 golang 提供的对象池实现，主要用于提高对象的复用效率，减少 GC。可以很简单地实现一个字节池，如下：

```GO
pool := &sync.Pool{
    New: func() interface{} {
        return make([]byte, 256)
    },
}
```

上面的字节池的缺点是：
-	每个资源大小都是固定的，有些场景不需要这么多的内存
-	出现非常大的数据时，会导致 `[]byte` 自动扩容，再放回池子中会占用很大内存，影响gc效率

如何解决上述问题呢？这就引入本文的内容。fasthttp 也提供了类似的实现 [bytebufferpool](https://github.com/valyala/bytebufferpool)，bytebufferpool 维护了一个数据对象池，比如当接收到 `http.Request` 的时候，就会从该对象池中获取一个数据对象填充，用完再归还。`bytebufferpool`的特点如下：

-	按照数据大小，对字节池分组，不同长度的数据放在能容纳的最小组里
-	占用内存过大的 `[]byte` 禁止放回池内，直接由os进行处理

本文基于`v1.0.0`版本分析下该库的实现

##  0x01	基本使用
Go 标准库中的类型`bytes.Buffer`封装字节切片，提供一些使用接口。由于切片的容量是有限的，容量不足时需要进行扩容。而频繁的扩容容易造成性能抖动。`bytebufferpool`实现了自己的Buffer类型，并使用一个简单的算法降低扩容带来的性能损失，使用方式如下，可以直接使用[默认对象池](https://github.com/valyala/bytebufferpool/blob/master/pool.go#L35)或者新建（可以根据实际需要创建新的`bytebufferpool`，将相同用处的对象放在一起）

```go
func main() {
  //使用默认的对象池
  b := bytebufferpool.Get()	
  b.WriteString("hello")
  b.WriteByte(',')
  b.WriteString(" world!")

  fmt.Println(b.String())

  bytebufferpool.Put(b)

  //业务场景1：新建
    joinPool := new(bytebufferpool.Pool)
	c := joinPool.Get()
	c.WriteString("hello")
	c.WriteByte(',')
	c.WriteString(" world!")

	fmt.Println(c.String())

    //使用完了归还
	joinPool.Put(c)

}
```

典型的使用方式先通过`bytebufferpool`提供的`Get()`方法获取一个`bytebufferpool.Buffer`对象，然后调用这个对象的方法写入数据，使用完成之后再调用`bytebufferpool.Put()`将对象放回对象池中



##	0x02    数据结构


####    ByteBuffer
[ByteBuffer](https://github.com/valyala/bytebufferpool/blob/master/bytebuffer.go)是对`[]byte`的封装，并实现了`io.Reader` 和 `io.Writer` 接口

```go
// ByteBuffer provides byte buffer, which can be used for minimizing
// memory allocations.
//
// ByteBuffer may be used with functions appending data to the given []byte
// slice. See example code for details.
//
// Use Get for obtaining an empty byte buffer.
type ByteBuffer struct {

	// B is a byte buffer to use in append-like workloads.
	// See example code for details.
	B []byte
}
```

作者实现`ByteBuffer`结构的目的是什么呢？


####    Pool
核心结构 [Pool](https://github.com/valyala/bytebufferpool/blob/master/pool.go#L25) 定义如下：
```golang
// Pool represents byte buffer pool.
//
// Distinct pools may be used for distinct types of byte buffers.
// Properly determined byte buffer types with their own pools may help reducing
// memory waste.
type Pool struct {
	calls       [steps]uint64
	calibrating uint64

	defaultSize uint64  
	maxSize     uint64  //允许放入pool池中的最大对象大小，只有<maxSize 的对象才允许放放池中

	pool sync.Pool      //使用了标准库中的对象sync.Pool
}
```

-   `calls`：缓存对象大小调用次数统计，steps 就是我们上面定义的常量。主要用来统计每类缓存大小的调用次数。steps 具体的值会使用一个index() 函数通过位操作的方式计算出来它在这个数组的索引位置；
-   `calibrating`：校标标记，0 表示未校准，1表示正在校准。校准完成需要从1恢复到0
-   `defaultSize`：缓存对象默认大小。我们知道当从 pool 中获取缓存对象时，如果池中没有对象可取，会通过调用 一个 New() 函数创建一个新对象返回，这时新创建的对象大小为 defaultSize。当然这里没有使用New() 函数,而是直接创建了一个 指定默认大小的 ByteBuffer；


####    内部结构callSize
主要是排序用，其中的`index`方法用来计算当前`size`落在哪个区间

```GO
type callSize struct {
	calls uint64
	size  uint64
}

type callSizes []callSize

func (ci callSizes) Len() int {
	return len(ci)
}

func (ci callSizes) Less(i, j int) bool {
	return ci[i].calls > ci[j].calls
}

func (ci callSizes) Swap(i, j int) {
	ci[i], ci[j] = ci[j], ci[i]
}

func index(n int) int {
	n--
	n >>= minBitSize
	idx := 0
	for n > 0 {
		n >>= 1
		idx++
	}
	if idx >= steps {
		idx = steps - 1
	}
	return idx
}
```


##  0x03    Get And Put 分析

```GOLANG
// Get returns an empty byte buffer from the pool.
func Get() *ByteBuffer { return defaultPool.Get() }

// Put returns byte buffer to the pool.
func Put(b *ByteBuffer) { defaultPool.Put(b) }
```

####    `pool.Get`方法

该方法比较直观，如下步骤：

```GO
// Get returns new byte buffer with zero length.
//
// The byte buffer may be returned to the pool via Put after the use
// in order to minimize GC overhead.
func (p *Pool) Get() *ByteBuffer {
    v := p.pool.Get()
    if v != nil {
        return v.(*ByteBuffer)
    }
    return &ByteBuffer{
        B: make([]byte, 0, atomic.LoadUint64(&p.defaultSize)),
    }
}
```

1.  直接从 `p.pool` 中调用内置 `Get()` 读取，如果不为`nil`，则说明当前池中读取到了数据，则直接返回对象 `*ByteBuffer` 即可
2.  如果结果为`nil` ，则说明池中已无对象可用且未定义`New`方法，这时直接创建一个 `p.defaultSize` 大小的 `*ByteBuffer` 对象并返回

这样的容量能满足绝大多数情况下的使用，避免使用过程中的切片扩容（但是不能完全避免扩容的case出现）

####    `pool.Put`方法
`Put`方法实现如下，这里着重分析下其优化的细节：在将对象放回池中时，会根据当前切片的容量进行相应的处理。`bytebufferpool`将大小分为 `20` 个[steps（区间）](https://github.com/valyala/bytebufferpool/blob/master/pool.go#L11)，类似于slab内存池算法的分级

区间如下，根据当前切片的实际容量落入下面的区间中：

-   小于`2^6`size
-   `2^6` ~ `2^7-1`
-   `......`
-   大于`2^25` size

执行足够多的`Put`后，bytebufferpool会重新校准，计算处于哪个区间容量的对象最多。将`defaultSize`设置为该区间的上限容量，上限容量即为右闭区间的值，即第一个区间的上限容量为 `2^6`，最后一个区间为 `2^26`。

通过`Get()`请求对象时，若池中无空闲对象，创建一个新对象时，直接将容量设置为`defaultSize`。这样基本可以避免在使用过程中的切片扩容，从而提升性能（预测使用最多的size为最小的size，是一种折中方案，可能会存在内存浪费的情况）

全局变量的定义：
```GO
const (
	minBitSize = 6 // 2**6=64 is a CPU cache line size
	steps      = 20

	minSize = 1 << minBitSize
	maxSize = 1 << (minBitSize + steps - 1)

	calibrateCallsThreshold = 42000 //触发校准的经验值
	maxPercentile           = 0.95
)
```

`Put`方法的代码：
```GO
// Put releases byte buffer obtained via Get to the pool.
//
// The buffer mustn't be accessed after returning to the pool.
func (p *Pool) Put(b *ByteBuffer) {
    // 对象在数据中的位置（落在哪个区间）
    idx := index(len(b.B))  

    // 首先计算切片容量落在哪个区间，增加calls数组中相应元素的值
    if atomic.AddUint64(&p.calls[idx], 1) > calibrateCallsThreshold {
        p.calibrate()
    }

    // 是否需要放入pool池中，还是直接丢弃交给GC
    // 如果要放回的对象容量大于 maxSize，则不放回
    maxSize := int(atomic.LoadUint64(&p.maxSize))
    if maxSize == 0 || cap(b.B) <= maxSize {
        b.Reset()
        p.pool.Put(b)
    }
}
```

如果calls数组该元素超过指定值`calibrateCallsThreshold=42000`（说明距离上次校准，放回对象到该区间的次数已经达到阈值了），则调用`Pool.calibrate()`执行校准操作

####    calibrate方法
1.  使用`atomic`包的`CompareAndSwapUint64`和`StoreUint64`来实现互斥锁功能
2.  汇总`calls`数组，得到频率最高的切片长度
3.  修正得到`maxSize`的值

```golang
func (p *Pool) calibrate() {
    // 原子操作，控制并发，避免并发放回对象触发（避免加锁）
	if !atomic.CompareAndSwapUint64(&p.calibrating, 0, 1) {
		return
	}

    // step 1.统计callSizes数组并排序
	a := make(callSizes, 0, steps)
	var callsSum uint64
	for i := uint64(0); i < steps; i++ {
		calls := atomic.SwapUint64(&p.calls[i], 0)  //calls数组记录了放回对象到对应区间的次数。按照这个次数从大到小排序
		callsSum += calls
		a = append(a, callSize{
			calls: calls,
			size:  minSize << i,    //minSize << i表示区间i的上限容量
		})
	}
	sort.Sort(a)

     // step 2.计算 defaultSize 和 maxSize
	defaultSize := a[0].size    //defaultSize很好理解，取排序后的第一个size即可
	maxSize := defaultSize      

	maxSum := uint64(float64(callsSum) * maxPercentile)
	callsSum = 0
	for i := 0; i < steps; i++ {
		if callsSum > maxSum {
			break
		}
		callsSum += a[i].calls
		size := a[i].size
		if size > maxSize {
			maxSize = size
		}
	}

    //maxSize值记录放回次数超过 95% 的多个对象容量的最大值。它的作用是防止将使用较少的大容量对象放回对象池，从而占用太多内存

    // step 3.保存对应值
	atomic.StoreUint64(&p.defaultSize, defaultSize)
	atomic.StoreUint64(&p.maxSize, maxSize)
	atomic.StoreUint64(&p.calibrating, 0)
}
```

注意上面代码中对`defaultSize`和`maxSize`的理解：
-   `defaultSize`：取排序后的第一个size，即统计次数最多的值
-   `maxSize`：记录放回次数超过 `95%` 的多个对象容量的最大值，其作用是防止将使用较少的大容量对象放回对象池，从而占用太多内存（直接让os层面去GC好了）

上面的逻辑和标准库的`fmt`包实现异曲同工，参考[标准库 fmt 中的应用](https://pandaychen.github.io/2020/03/11/GOLANG-SYNC-POOL-USAGE/#%E6%A0%87%E5%87%86%E5%BA%93-fmt-%E4%B8%AD%E7%9A%84%E5%BA%94%E7%94%A8)

##  0x04    应用
最常见的应用场景是解析http响应时：
```go
func doHttpRequest(){
    //...
    resp, err := client.Do(req)

    if err!=nil{
        return
    }

	defer resp.Body.Close()

	ret.HTTPCode = resp.StatusCode
	if resp.StatusCode != http.StatusOK {
		err = fmt.Errorf("http code is %d", resp.StatusCode)
		return
	}

    //
	b := bytebufferpool.Get()
	_, err = b.ReadFrom(resp.Body)
	if err != nil {
		return
	}

	l := b.Len()
	if l < 0 {
		return
	}

    //使用后归还
	ret.Response = b.Bytes()
    bytebufferpool.Put(b)

    // ....
}
```


##  0x05 总结
本文分析了bytebufferpool的实现细节：

-   相对于标准库`sync.Pool`更优的点，基于切片的长度实现的精细化回收复用池
-   容量**最小值**取 `2^6 = 64`（是 `64` 位计算机上 CPU 缓存行的大小，可以一次性被加载到 CPU 缓存行）
-   使用`atomic`而不是锁来实现并发控制，可借鉴

##  0x06	参考
-   [Anti-memory-waste byte buffer pool：bytebufferpool](https://github.com/valyala/bytebufferpool)
-	[深入理解高性能字节池 bytebufferpool](https://blog.csdn.net/why444216978/article/details/122016389)
-   [缓存池 bytebufferpool 库实现原理](https://www.jianshu.com/p/a730d095ae51)