---
layout:     post
title:      Sync.Pool 应用与分析（续）
subtitle:   Golang sync.Pool 源码分析
date:       2023-05-01
author:     pandaychen
catalog:    true
tags:
    - Golang
    - Pool
---


##  0x00    前言

前文 [Golang 中的 sync.Pool 使用](https://pandaychen.github.io/2020/03/11/GOLANG-SYNC-POOL-USAGE/) 介绍了 `sync.Pool` 的使用及若干细节。

`sync.Pool` 是并发安全且可伸缩的，常用于存放可复用的对象的一个容器，以减少反复创建对象带来的开销，这里需要注意，Pool 是用于存放对象的，不建议用来缓存一些有状态的对象（如长连接等）或数据等持久存储对象，因为 Pool 中的内容是会随着 GC 而被回收。

当项目开发中，如果发现 GC 耗时很高，有大量临时对象时不妨可以考虑使用 `sync.Pool`

源码的开头介绍：

> An appropriate use of a Pool is to manage a group of temporary items
> silently shared among and potentially reused by concurrent independent
> clients of a package. Pool provides a way to amortize allocation overhead
> across many clients.

`sync.Pool` 适合于管理需要重复使用的对象，这些对象由 golang 的垃圾回收 GC 管理，无需调用者介入。当 GC 需要将其中部分对象进行回收时，不会告知使用者。使用者也无从得知当前 Pool 中管理有多少对象，并且每次 `Put` 和 `Get` 操作都是随机获得的对象。

本文以 `go1.16` 为例，分析下 `sync.Pool` 的 [源码实现](https://github.com/golang/go/blob/master/src/sync/pool.go)

####    正确姿势
再着重 mark 下正确使用姿势：

1.  设置 `New` 方法
2.  使用时直接 `Get`
3.  使用完成后先进行字段清空, 然后在 `Put` 放回（一定要进行 `Reset`，否则可能会有意外问题）

##  0x01    源码分析


####    核心数据结构

还记得 golang 经典的 GMP 调度模型吗？在 GMP 调度模型中，`M` 代表了系统线程，而同一时间一个 `M` 上只能同时运行一个 `P`。那么也就意味着，从线程维度来看，在 `P` 上的逻辑都是单线程执行的。而 `sync.Pool` 充分利用了 GMP 这一特点：对于同一个 `sync.Pool` ，每个 `P` 都有一个自己的本地对象池 `poolLocal`（且不需要加锁访问）；通过这样的设计，每个 `P` 都有了自己的本地空间，多个 goroutine 使用同一个 Pool 时，减少了竞争，提升了性能

pool 结构如下图所示：
![oool-struct](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/pool/syncpool/pool-struct.png)

![pool-struct-1](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/pool/syncpool/pool-arch-1.png)

####    Pool 结构

```GO
type Pool struct {
    // 用于检测 Pool 池是否被 copy，因为 Pool 不希望被 copy，可用用 go vet 工具检测，在编译期间就发现问题
	noCopy noCopy

    // 数组结构，对应每个 P，数量和 P 的数量一致
	local     unsafe.Pointer // local fixed-size per-P pool, actual type is [P]poolLocal
	localSize uintptr        // size of the local array

    // GC 到时，victim 和 victimSize 会分别接管 local 和 localSize
    // victim 的目的是为了减少 GC 后冷启动导致的性能抖动，让分配对象更平滑
	victim     unsafe.Pointer // local from previous cycle
	victimSize uintptr        // size of victims array

    // 对象初始化构造方法，使用方定义
	New func() interface{}
}
```

-   `local` 和 `localSize` 实现了一个数组，数组元素为 `poolLocal` 结构，用来管理临时对象；数组中的每个节点都对应每个 `P`，数量和 `P` 的数量一致
-   `victim` 和 `victimSize`  是在 `poolCleanup` 流程里赋值了（GC 到时，`victim` 和 `victimSize` 会分别接管 `local` 和 `localSize`），赋值的内容就是 `local` 和 `localSize` 。`victim` 机制是把 Pool 池的清理由一轮 GC 改成 两轮 GC，进而提高对象的复用率，减少 GC 带来的抖动

额外说一下 `victim` 的 [作用](https://www.cyhone.com/articles/think-in-sync-pool/)：

TODO

####    poolLocal 结构

```GO
// Pool.local 指向的数组元素类型
type poolLocal struct {
  poolLocalInternal

  // 把 poolLocal 填充至 128 字节对齐，避免 false sharing 引起的性能问题
  pad [128 - unsafe.Sizeof(poolLocalInternal{})%128]byte
}

// 管理 cache 的内部结构，跟每个 P 对应，操作无需加锁
type poolLocalInternal struct {
    // 每个 P 的私有，使用时无需加锁
    private interface{}         // 注意：单个对象
    // 双链表结构，用于挂接 cache 元素
    shared  poolChain       // 双向链表
}

type poolChain struct {
  head *poolChainElt
  tail *poolChainElt
}
```

`poolLocal` 是管理 Pool 池里 cache 元素的关键结构，`Pool.local` 指向的就是这么一个类型的数组，该结构值得注意的一点是使用了内存填充，对齐 cache line，防止 false sharing 性能问题的技巧。Pool 里面该结构数组是按照 `P` 的个数分配的，每个 `P` 都对应一个这个结构（见上图）

`poolLocalInternal` 的定义，其中每个本地对象池，都会包含两项：
-   `private`：私有变量，只能由相应的一个 `P` 存取。`Get` 和 `Put` 操作都会优先存取 `private` 变量，如果 `private` 变量可以满足情况，则不再深入进行其他的复杂操作；每个 `P` 都有，用于不同 `G` 执行 `get` 和 `put` 可以无锁操作；一个 `P` 同时只能执行一个 goroutine，所以不会有并发的问题
-   `shared`：这就是 `P` 的本地对象池，思考一下为啥本地的 `P` 取操作也要加锁？（共享对象数组，每个 `P` 都有一个，同一个 `P` 上不同 `G` 可以多次执行 `put` 方法，需要有地方能存储。并且别的 `P` 上的 `G` 可能过来偷，所以要加锁）

注意注意：`shared` 可以由任意的 `P` 访问，但是只有本地的 `P` 才能 `pushHead/popHead`，其它 `P` 可以 `popTail`（这几个操作都是 `poolChain` 提供的方法），`poolChain` 是一个双端队列，里面的 `head` 和 `tail` 分别指向队列头尾；`poolDequeue` 里面存放真正的数据，是一个单生产者、多消费者的固定大小的无锁的环状队列，`headTail` 是环状队列的首位置指针，可以通过位运算解析出首尾的位置，生产者（对应上面本地的 `P`）可以从 `head` 插入、`head` 删除，而消费者（对应上面其他 `P`）仅可从 `tail` 删除

####    poolChain && poolChainElt
`poolChainElt` 是双链表的元素，里面其实是一段数组空间，类似于 ringbuffer，Pool 管理的 cache 对象就都存储在 `poolDequeue` 结构的 `vals[]` 数组中

```GO
type poolChain struct {
  head *poolChainElt
  tail *poolChainElt
}

type poolChainElt struct {
  // 本质是个数组内存空间，管理成 ringbuffer 的模式；
  poolDequeue
  // 链表指针
  next, prev *poolChainElt
}

type poolDequeue struct {
  headTail uint64

  // vals is a ring buffer of interface{} values stored in this
  // dequeue. The size of this must be a power of 2.
  vals []eface
}
```

`poolChain` 及 `poolDequeue` 结构如下图，被缓存的临时对象都存储在各个环的 `vals` 中：
![pool-dequeue](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/pool/syncpool/pool-chain-arch.png)


####  Get 对外接口
`sync.Pool` 内部调用 `Get` 获取对象时，先尝试从当前 `P` 的 poolLocal private 中获取（同一时间 `P` 只有一个 goroutine 运行，且 private 为该 `P` 私有，因而不用加锁），如果未获取到，则尝试从自身的 `shared` 共享列表中查询可用的对象（存在同其他 goroutine 竞争访问，需要加锁），如果还没找到则只能去其他 `P` 中的 `shared` 偷了

```GO
func (p *Pool) Get() interface{} {
    if race.Enabled {
        race.Disable()
    }
    // 把 G 锁住在当前 M（声明当前 M 不能被抢占），返回 M 绑定的 P 的 ID
    // 在当前的场景，也可以认为是 G 绑定到 P，因为这种场景 P 不可能被抢占，只有系统调用的时候才有 P 被抢占的场景
    l := p.pin()
    x := l.private// 尝试从 P 中 poolLocal 的 private 获取，如果能从 private 取出缓存的元素，那么将是最快的路径
    l.private = nil
    runtime_procUnpin()
    if x == nil {
        l.Lock()// 尝试从 P 本地 poolLocal 的 shared 中获取
        // 从 shared 队列里获取，shared 队列在 Get 获取，在 Put 投递
        last := len(l.shared) - 1
        if last >= 0 {
            x = l.shared[last]
            l.shared = l.shared[:last]
        }
        l.Unlock()
        if x == nil {// 尝试从其他 P 中 poolLocal 的 shared 中获取
            // 尝试从获取其他 P 的队列里取元素，或者尝试从 victim cache 里取元素
            x = p.getSlow()
        }
    }
    if race.Enabled {
        race.Enable()
        if x != nil {
            race.Acquire(poolRaceAddr(x))
        }
    }
    // 还是没有取到，New 一个
    // 最慢的路径：现场初始化，这种场景是 Pool 池里一个对象都没有，只能现场创建
    if x == nil && p.New != nil {
        x = p.New()
    }
    return x
}
```

小结下 `Get` 方法的流程：
1.  `Get` 操作时，先返回本地 `P` 上的 `private` 上的对象
2.  如果 `private` 为空，继续从本地 `P` 上的 `shared` 找，这里需要加锁
3.  如果 `shared` 也没有，就到别的 `P` 那儿，从 `shared` 里偷
4.  所有其它 `P` 都遍历过了，没有任何对象可偷。就返回 `nil` 或调用 `New` 函数新建

####    getSlow 方法

####    Put对外接口
`Put` 则相对来说比较简单，优先放入自身的 `private`，如果不为空则追加到自身的 `shared` 后面
```go
// Put adds x to the pool.
func (p *Pool) Put(x interface{}) {
    if x == nil {
        return
    }
    if race.Enabled {
        if fastrand()%4 == 0 {
            // Randomly drop x on floor.
            return
        }
        race.ReleaseMerge(poolRaceAddr(x))
        race.Disable()
    }
    l := p.pin()
    if l.private == nil {
        l.private = x
        x = nil
    }
    runtime_procUnpin()
    if x != nil {
        l.Lock()
        l.shared = append(l.shared, x)
        l.Unlock()
    }
    if race.Enabled {
        race.Enable()
    }
}
```

##  0x02  总结
`sync.Pool` 中的资源随时都有可能被销毁而消失



##  0x03 参考
-   [多图详解 Go 的 sync.Pool 源码](https://www.luozhiyun.com/archives/416)
-   [深度分析 Golang sync.Pool 底层原理](https://www.cyhone.com/articles/think-in-sync-pool/)
-   [Runtime: Golang 之 sync.Pool 源码分析](https://blog.haohtml.com/archives/24697)
-   [Go 语言之 sync.pool 源码分析](https://www.lixueduan.com/posts/go/sync-pool/)
-   [Go sync.Pool 浅析](https://segmentfault.com/a/1190000040029288)
-   [Go 并发编程 — 深度剖析 sync.Pool 源码级原理](https://xie.infoq.cn/article/55f28d278cccf0d8195459263)
-   [sync.Pool](https://cs.opensource.google/go/go/+/go1.17.13:src/sync/pool.go;l=124)
-   [What’s false sharing and how to solve it (using Golang as example)](https://medium.com/@genchilu/whats-false-sharing-and-how-to-solve-it-using-golang-as-example-ef978a305e10)