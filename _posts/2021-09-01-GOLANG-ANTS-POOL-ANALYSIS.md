---
layout: post
title: Golang 并发协程池实现分析（三）
subtitle: 分析 Ants 协程池实现
date: 2021-09-01
author: pandaychen
header-img: img/golang-horse-fly.png
catalog: true
category: false
tags:
  - Golang
---

## 0x00 前言

本文分析下协程池库 ants 的实现，仓库 [在此](https://github.com/panjf2000/ants)，此库基于 fasthttp 的协程池实现。

## 0x01 ants 协程池使用

ants 提供了两种执行模式：
1、`ants.NewPool(pool_size)`<br>
通过这种方式创建的 Pool，需要调用 `pool.Submit(task)` 提交任务，任务是一个无参数无返回值的函数，适合不关注结果的并发任务场景
2、`ants.NewPoolWithFunc(pool_size, func(interface{}))`<br>
这种方式创建的 Pool 需要指定任务处理函数，需调用 `p.Invoke(arg)` 提交任务，`arg` 是传递给 `func(interface{})` 的参数，此 Pool 适合关注结果的并发任务执行场景

## 0x02 ants 分析

ants 的运行流程图如下，比较直观，我们按照如下几个核心模块进行分析：

- Pool：协程池的核心结构，一个 Pool 一般生成固定个 Worker
- Worker：ants 中为每个任务都是由 worker 对象来处理的，每个 worker 对象会对应创建一个 goroutine 来处理任务，一个 worker 对应于一个 goroutine
- Task：用户指定的运行方法，即单个任务

![img]()

## 0x03 Pool 分析

#### Pool 结构定义

Pool 的结构体 [定义](https://github.com/panjf2000/ants/blob/master/pool.go#L34) 如下：

```golang
// Pool accepts the tasks from client, it limits the total of goroutines to a given number by recycling goroutines.
type Pool struct {
	// capacity of the pool, a negative value means that the capacity of pool is limitless, an infinite pool is used to
	// avoid potential issue of endless blocking caused by nested usage of a pool: submitting a task to pool
	// which submits a new task to the same pool.
	capacity int32  //ants 最多能创建的 goroutine 数量

	// running is the number of the currently running goroutines.
	running int32   // 已经创建的 worker goroutine 的数量

	// lock for protecting the worker queue.
	lock sync.Locker    //ants 自己实现了一个自旋锁。用于同步并发操作

	// workers is a slice that store the available workers.
	workers workerArray // 存放一组 worker 对象，即一组 goroutine，workerArray 只是一个 interface，存放 goWorker 对象的容器（见下文分析）

	// state is used to notice the pool to closed itself.
	state int32     // 记录池子当前的状态，是否已关闭（CLOSED）

	// cond for waiting to get a idle worker.
	cond *sync.Cond // 条件变量，处理任务等待和唤醒

	// workerCache speeds up the obtainment of a usable worker in function:retrieveWorker.
	workerCache sync.Pool   // 使用 sync.Pool 对象池管理和创建 worker 对象，提升性能

	// blockingNum is the number of the goroutines already been blocked on pool.Submit, protected by pool.lock
	blockingNum int     // 当前阻塞等待的任务数量

	options *Options
}

type workerArray interface {
	len() int
	isEmpty() bool
	insert(worker *goWorker) error
	detach() *goWorker
	retrieveExpiry(duration time.Duration) []*goWorker
	reset()
}
```

注意 `workerArray` 类型是一个抽象类型 `interface{}`，ants 提供了基于 `stackType` 和 `loopQueueType` 的两种 [实现](https://github.com/panjf2000/ants/blob/master/worker_array.go#L32)。`workerArray` 中的核心结构是 [goWorker](https://github.com/panjf2000/ants/blob/master/worker.go#L33)。通常对于协程池，一个 `Pool` 会生成固定的若干个 `goWorker`，对任务分配指定的 `goWorker` 来实现流水线任务运行，从而达到复用 worker 的目的。

```golang
// goWorker is the actual executor who runs the tasks,
// it starts a goroutine that accepts tasks and
// performs function calls.
type goWorker struct {
	// pool who owns this worker.
	pool *Pool

	// task is a job should be done.
	task chan func()

	// recycleTime will be updated when putting a worker back into queue.
	recycleTime time.Time
}
```

#### 创建 Pool 的方式

[NewPool](https://github.com/panjf2000/ants/blob/master/pool.go#L97) 方法如下，注意 `p.workerCache` 及 `p.workers` 的初始化，此外在 `NewPool` 中还创建了子协程 `purgePeriodically()` 用于定时回收割超时的 worker：

```golang
// NewPool generates an instance of ants pool.
func NewPool(size int, options ...Option) (*Pool, error) {
	opts := loadOptions(options...)

	if size <= 0 {
		size = -1
	}

	if expiry := opts.ExpiryDuration; expiry < 0 {
		return nil, ErrInvalidPoolExpiry
	} else if expiry == 0 {
		opts.ExpiryDuration = DefaultCleanIntervalTime
	}

	if opts.Logger == nil {
		opts.Logger = defaultLogger
	}
    // 创建 Pool 对象
	p := &Pool{
		capacity: int32(size),
		lock:     internal.NewSpinLock(),
		options:  opts,
	}
	p.workerCache.New = func() interface{} {
		return &goWorker{
			pool: p,
			task: make(chan func(), workerChanCap),
		}
	}
	if p.options.PreAlloc {
		if size == -1 {
			return nil, ErrInvalidPreAllocSize
		}
		// 预先分配固定 Size 的池子
		p.workers = newWorkerArray(loopQueueType, size)
	} else {
		// 初始化不创建，运行时再创建
		p.workers = newWorkerArray(stackType, 0)
	}

	p.cond = sync.NewCond(p.lock)

	// Start a goroutine to clean up expired workers periodically.
	go p.purgePeriodically()

	return p, nil
}
```

此外，在创建 Pool 时，根据 `options.PreAlloc` 设置，分两种方式（上面代码）：

- 预先分配
- 运行时创建

调用创建 Pool 的方法时，创建 Pool 的类型又会分为 Stack 及 Queue 两种结构，见下文的分析。

#### 销毁 Pool

```golang
func (p *Pool) purgePeriodically() {
  heartbeat := time.NewTicker(p.options.ExpiryDuration)
  defer heartbeat.Stop()

  for range heartbeat.C {
    if p.IsClosed() {
      break
    }

    p.lock.Lock()
    expiredWorkers := p.workers.retrieveExpiry(p.options.ExpiryDuration)
    p.lock.Unlock()

    for i := range expiredWorkers {
      expiredWorkers[i].task <- nil
      expiredWorkers[i] = nil
    }

    if p.Running() == 0 {
      p.cond.Broadcast()
    }
  }
}
```

如何触发某个 goroutine 主动退出呢？

#### 向 Pool 中提交 Task

Pool 提供了 `Submit` 方法，提供外部发起任务调度的接口：

```golang
func (p *Pool) Submit(task func()) error {
	// 判断 pool 是否关闭
  if p.IsClosed() {
    return ErrPoolClosed
  }
  var w *goWorker
  //retrieveWorker 方法获取空闲的 worker
  if w = p.retrieveWorker(); w == nil {
    return ErrPoolOverload
  }

  // 将待执行的任务 send 到 goWorker 的 任务 channel 中
  w.task <- task
  return nil
}
```

#### Pool 获取可用的 goWoker

通过 `retrieveWorker` 方法获取当前池中可用的（空闲的） `goWorker`，该方法实现了开头示意图的逻辑：

```golang
func (p *Pool) retrieveWorker() (w *goWorker) {
		//spawnWorker 方法
		spawnWorker := func() {
			w = p.workerCache.Get().(*goWorker)
			w.run()
		}
        p.lock.Lock()

        //1. 从 workers 中取出一个 goWorker
        //p.workers 是 loopQueue 或者 workerStack 对象，它们都实现了 detach() 方法
        w = p.workers.detach()
        if w != nil {
			//SUCC，有空闲 goroutine，直接返回
            p.lock.Unlock()
        } else if capacity := p.Cap(); capacity == -1 || capacity> p.Running() {
			// 池容量还没用完（即容量大于正在工作的 goWorker 数量），则调用 spawnWorker() 新建一个 goWorker，执行其 run() 方法，直接返回
			p.lock.Unlock()
			spawnWorker()
        } else {
			if p.options.Nonblocking {
				// 设置了非阻塞选项，直接返回 nil
					p.lock.Unlock()
					return
			}
        RETRY:
                if p.options.MaxBlockingTasks != 0 && p.blockingNum >= p.options.MaxBlockingTasks {
					// 如果设置了最大阻塞队列长度限制，并且当前阻塞等待的任务数量已经达到这个上限，直接返回 nil
                        p.lock.Unlock()
                        return
                }
				// 未超过最大阻塞队列长度限制
                p.blockingNum++
				// 调用 p.cond.Wait() 等待
				// 注意！这里会阻塞，通过 p.cond.Signal() 方法会唤醒这里的逻辑
                p.cond.Wait()
				// 阻塞的任务数减 1
                p.blockingNum--
                var nw int
                if nw = p.Running(); nw == 0 {
                        p.lock.Unlock()
                        if !p.IsClosed() {
                                spawnWorker()
                        }
                        return
                }
                if w = p.workers.detach(); w == nil {
                        if nw < capacity {
                                p.lock.Unlock()
                                spawnWorker()
                                return
                        }
                        goto RETRY
                }

                p.lock.Unlock()
        }
    return
}
```

## 0x04 Worker 实现

ants 中为每个任务都是由 worker 对象来处理的，每个 worker 对象会对应创建一个 goroutine 来处理任务，Worker 对应的结构是 [goWorker](https://github.com/panjf2000/ants/blob/master/worker.go#L33)，其中 `recycleTime` 标识了空闲开始时间，该字段只在非 `PreAlloc` 模式（运行时创建模式）下才起效

```golang
// goWorker is the actual executor who runs the tasks,
// it starts a goroutine that accepts tasks and
// performs function calls.
type goWorker struct {
	// pool who owns this worker.
	pool *Pool	// 指向属主池

	// task is a job should be done.
	task chan func()	// 非常重要！任务通道，通过这个通道将类型为 func () 的函数作为任务发送给 goWorker 执行

	// recycleTime will be updated when putting a worker back into queue.
	recycleTime time.Time	// 此字段记录 goroutine 放回池中的时间（即什么时候开始空闲），注意：此字段只在非 PreAlloc 模式下才起效
}
```

goWorker 的核心是 `run` 方法，该方法启动一个子 goroutine，然后不停地从 `task` 通道中接收任务，然后执行任务 `f()`，任务执行完成之后调用 `Pool.revertWorker` 方法将该 `goWorker` 对象放回 Pool 中，以便下次取出处理新的任务；

PS：这里其实可以修改为 goroutine 一直不停的监听在 `w.task` 上，不必放回池中？

```golang
// run starts a goroutine to repeat the process
// that performs the function calls.
func (w *goWorker) run() {
	w.pool.incRunning()
	go func() {
		// 异常处理！
		defer func() {
			w.pool.decRunning()
			w.pool.workerCache.Put(w)
			if p := recover(); p != nil {
				if ph := w.pool.options.PanicHandler; ph != nil {
					ph(p)
				} else {
					w.pool.options.Logger.Printf("worker exits from a panic: %v\n", p)
					var buf [4096]byte
					n := runtime.Stack(buf[:], false)
					w.pool.options.Logger.Printf("worker exits from panic: %s\n", string(buf[:n]))
				}
			}
			// Call Signal() here in case there are goroutines waiting for available workers.
			w.pool.cond.Signal()
		}()

		for f := range w.task {
			//for loop here......
			if f == nil {
				return
			}
			f()
			if ok := w.pool.revertWorker(w); !ok {
				return
			}
		}
	}()
}
```

## 0x05 workerArray 实现

前面描述过，`workerArray` 是一个抽象的接口，实现了 <font color="#dd0000">goroutine 池的核心逻辑 </font>，ants 提供了 `workerStack` 和 `loopQueue` 两种实现：

#### workerArray 定义

```golang
type workerArray interface {
  len() int
  isEmpty() bool
  insert(worker *goWorker) error    // 任务执行结束后，将相应的 worker （goroutine）放回 workerArray 中
  detach() *goWorker                // 从 workerArray 中取出一个 worker（goroutine）
  retrieveExpiry(duration time.Duration) []*goWorker    // 取出所有的过期 worker
  reset()
}
```

下面看下 `workerArray` 具体的实现。

#### loopQueue 实现

LoopQueue 是基于循环队列的 [实现](https://github.com/panjf2000/ants/blob/master/worker_loop_queue.go#L5)，

```golang
type loopQueue struct {
	items  []*goWorker
	expiry []*goWorker
	head   int
	tail   int
	size   int
	isFull bool
}
```

TODO：待补充

#### workerStack 实现

[workerStack 实现](https://github.com/panjf2000/ants/blob/master/worker_stack.go#L5)，结构包含 `items` 和 `expiry` 两个成员，workerArray 也是采用运行时创建的逻辑来创建 goroutine：

```golang
type workerStack struct {
  items  []*goWorker       // 当前可用（空闲的） worker
  expiry []*goWorker        // 过期的 worker
  size   int
}

// 非预先创建，采用运行时创建的逻辑，insert 始终是插入到末尾位置
func (wq *workerStack) insert(worker *goWorker) error {
  wq.items = append(wq.items, worker)
  return nil
}
```

`detach` 方法实现了当有任务带处理时，从 `workerArray` 中取出一个空闲的 goroutine（`goWorker`），根据 stack 后进先出的特点，这里始终返回最后一个位置的节点 `wq.items[l-1]`：

```golang
func (wq *workerStack) detach() *goWorker {
  l := wq.len()
  if l == 0 {
    return nil
  }

  w := wq.items[l-1]
  wq.items[l-1] = nil // avoid memory leaks
  wq.items = wq.items[:l-1]

  return w
}
```

当 goroutine 完成任务之后，Pool 池会将相应的 worker 放回 `workerStack`，调用 `workerStack.insert()` 直接将 goroutine 放回 `items` 中。

此外，通过 `retrieveExpiry` 方法，获取到当前过期的 worker 列表，由于 ants 库是按运行时创建，所以这里加入过期的逻辑来回收掉那些多余的 goroutine，该方法会被 `Pool.purgePeriodically` 方法中调用与处理：

`retrieveExpiry` 的回收过程包含如下几个部分：

1.  查找过期的 goroutines 位置
2.  回收到这部分 goroutine
3.  未过期的 goroutine 转移到 `wq.items` 首部

![img]()

```golang
// 二分查找的是最近过期的 worker，即将过期的 worker 的前一个位置，在此位置之前的 worker 已经全部过期了
func (wq *workerStack) binarySearch(l, r int, expiryTime time.Time) int {
  var mid int
  for l <= r {
    mid = (l + r) / 2
    //expiryTime < recycleTime
    if expiryTime.Before(wq.items[mid].recycleTime) {
      r = mid - 1
    } else {
      l = mid + 1
    }
  }
  return r
}

func (wq *workerStack) retrieveExpiry(duration time.Duration) []*goWorker {
  n := wq.len()
  if n == 0 {
    return nil
  }

  // 获取超时的底限刻度
  expiryTime := time.Now().Add(-duration)
  // 采用二分法查找 index
  index := wq.binarySearch(0, n-1, expiryTime)

  //reset wq.expiry
  wq.expiry = wq.expiry[:0]
  if index != -1 {
    wq.expiry = append(wq.expiry, wq.items[:index+1]...)    // 在 0:index+1 区间的 goroutine 都需要被回收

    // 将 wq.items[index+1:] 部分的 goroutine 重新放回 wq.items（这些元素都是未过期的）
    m := copy(wq.items, wq.items[index+1:])
    for i := m; i < n; i++ {
        //reset origin zone
      wq.items[i] = nil
    }
    // 重新生成 wq.items
    wq.items = wq.items[:m]
  }
  return wq.expiry
}
```

思考一下，这里为何采用二分法进行搜索呢？二分法查找需要数组有序，而在 `workerStack` 中，由于过期时间是按照 goroutine 执行任务后的空闲时间计算的，而 `workerStack.insert` 方法的插入顺序正好满足这个特性，`wq.item` 中各个元素的过期时间是从早到晚有序的。

## 0x06 Options 选项

ants 提供了一些选项用于定制 goroutine Pool：

```golang
type Options struct {
	ExpiryDuration time.Duration	// 过期时间，表示 goroutine 空闲多长时间之后会被 ants 池回收
	PreAlloc bool	// 是否预分配
	MaxBlockingTasks int	// 最大阻塞任务数量，即池中 goroutine 数量已到池容量，且所有 goroutine 都处于繁忙状态，这时到来的任务会在阻塞列表等待。阻塞的任务数量达到这个值后，后续任务提交直接返回失败
	Nonblocking bool	// 阻塞开关，提交任务时，如果 ants 池中 goroutine 已到上限且全部繁忙，阻塞的池会将任务添加的阻塞列表等待（受限于阻塞列表长度）。非阻塞下直接返回失败
	PanicHandler func(interface{})	//panic 钩子方法，遇到 panic 会调用这里设置的处理函数
	Logger Logger
}
```

## 0x07 一些细节

#### slice 指针的回收

在 workerStack 的 `detach` 方法中，由于切片的底层结构是数组，只要有引用数组的指针，数组中的元素就不会释放。这里取出切片最后一个元素后，将对应数组元素的指针设置为 `nil`，主动释放这个引用。这也是防止内存泄漏的好办法。

```golang
func (wq *workerStack) detach() *goWorker {
  l := wq.len()
  if l == 0 {
    return nil
  }

  w := wq.items[l-1]
  wq.items[l-1] = nil // avoid memory leaks
  wq.items = wq.items[:l-1]

  return w
}
```

## 0x08 参考

- [Go 每日一库之 ants](https://darjun.github.io/2021/06/03/godailylib/ants/)
