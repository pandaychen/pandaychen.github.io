---
layout:     post
title:      GoZero 组件分析：executors（批处理组件）
subtitle:
date:       2024-01-15
author:     pandaychen
catalog:    true
tags:
    - GoZero
---


##  0x00 前言
现网中可能会遇到如下场景，批量提交一批任务（如 MYSQL 语句），或是从 kafka 等队列中批量消费一批数据；Go 的现有库没有诸如灵活定义作业运行、批量提交任务减少小任务提交等特性，当然可以借助于线程池来解决，不过 goZero 给出了一套更为轻量化的实现：[executors](https://github.com/zeromicro/go-zero/tree/master/core/executors)

本文分析下 [executors](https://github.com/zeromicro/go-zero/tree/master/core/executors) 组件实现，在 goZero 中，executors 充当任务池，做多任务缓冲，使用做批量处理的任务。如 clickhouse 大批量 insert，sql batch insert 等等。同时也可以在 [go-queue]() 也可以看到 executors （在 queue 里面使用的是 ChunkExecutor ，限定任务提交字节大小）

####    使用场景

-   批量提交（运行）任务
-   缓冲一部分任务，惰性（超过某个设定的条件，比如超过数量 OR 超时）提交
-   延迟任务提交


####    实现原理
以 `PeriodicalExecutor` 为例：
![period](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/gozero-tech/periodical-executor.png)

##  0x01    接口设计
executors 包实现了这几个 executor ：大部分都是 `executor` 配合 `container` 的组合设计
-   `bulkexecutor`：达到 `maxTasks` （最大任务数）提交
-   `chunkexecutor`	达到 `maxChunkSize`（最大字节数）提交
-   `periodicalexecutor`：最基础的 `executor` 实现
-   `delayexecutor`：延迟执行传入的 `fn()`
-   `lessexecutor`：特殊功能

##  0x02    源码分析

####   依赖关系
-   `bulkexecutor`：`periodicalexecutor` + `bulkContainer`
-   `chunkexecutor`：`periodicalexecutor` + `chunkContainer`

####   基础定义

1、`TaskContainer`（是一个 `interface`）：实例化为 `bulkContainer`、`chunkContainer`，可以理解为一个任务缓冲队列，`AddTask` 用来提示添加任务后是否超过了队列的最大值

```GO
type TaskContainer interface {
		// AddTask adds the task into the container.
		// Returns true if the container needs to be flushed after the addition.
        // 把 task 加入 container
		AddTask(task any) bool
		// Execute handles the collected tasks by the container when flushing.
        // 实际上是去执行传入的 execute func()
		Execute(tasks any)
		// RemoveAll removes the contained tasks, and return them.
        // 达到临界值，移除 container 中全部的 task，通过 channel 传递到 execute func() 执行
		RemoveAll() any
}
```

```GO
type bulkContainer struct {
	tasks    []any
	execute  Execute
	maxTasks int
}

func (bc *bulkContainer) AddTask(task any) bool {
	bc.tasks = append(bc.tasks, task)
	return len(bc.tasks) >= bc.maxTasks
}

func (bc *bulkContainer) Execute(tasks any) {
	vals := tasks.([]any)
	bc.execute(vals)
}

func (bc *bulkContainer) RemoveAll() any {
	tasks := bc.tasks
	bc.tasks = nil
	return tasks
}
```

```GO
type chunkContainer struct {
	tasks        []any
	execute      Execute
	size         int
	maxChunkSize int
}

func (bc *chunkContainer) AddTask(task any) bool {
	ck := task.(chunk)
	bc.tasks = append(bc.tasks, ck.val)
	bc.size += ck.size
	return bc.size >= bc.maxChunkSize
}

func (bc *chunkContainer) Execute(tasks any) {
	vals := tasks.([]any)
	bc.execute(vals)
}

func (bc *chunkContainer) RemoveAll() any {
	tasks := bc.tasks
	bc.tasks = nil
	bc.size = 0
	return tasks
}

type chunk struct {
	val  any
	size int
}
```


####    核心：PeriodicalExecutor 的实现
`PeriodicalExecutor`[定义](https://github.com/zeromicro/go-zero/blob/master/core/executors/periodicalexecutor.go#L32) 如下：
```GO
// A PeriodicalExecutor is an executor that periodically execute tasks.
type PeriodicalExecutor struct {
    commander chan any          // 异步执行
    interval  time.Duration
    container TaskContainer
    waitGroup sync.WaitGroup
    // avoid race condition on waitGroup when calling wg.Add/Done/Wait(...)
    wgBarrier   syncx.Barrier
    confirmChan chan lang.PlaceholderType
    inflight    int32
    guarded     bool
    newTicker   func(duration time.Duration) timex.Ticker
    lock        sync.Mutex
}

// 初始化
func New...(interval time.Duration, container TaskContainer) *PeriodicalExecutor {
    executor := &PeriodicalExecutor{
        commander:   make(chan interface{}, 1),
        interval:    interval,
        container:   container,
        confirmChan: make(chan lang.PlaceholderType),
        newTicker: func(d time.Duration) timex.Ticker {
            return timex.NewTicker(interval)
        },
    }
    //...
    return executor
}
```

`PeriodicalExecutor` 结构的重要成员定义如下：
-   `commander`：异步传递 `tasks` 的 channel（从 `container` 加入的 tasks）
-   `container`：暂存 `Add()` 增加的 task
-   `confirmChan`：阻塞 `Add()` ，在开始本次的 `executeTasks()` 会放开阻塞
-   `ticker`：定时器，防止 `Add()` 阻塞时（未达批量处理上限），会有一个定时执行的机会，及时释放暂存的 task

####    `PeriodicalExecutor.Add`方法

-   `Add`
-   `Flush`
-   `Sync`
-   `Wait`

```GO
// Add adds tasks into pe.
func (pe *PeriodicalExecutor) Add(task any) {
	if vals, ok := pe.addAndCheck(task); ok {
		pe.commander <- vals
		<-pe.confirmChan
	}
}

func (pe *PeriodicalExecutor) addAndCheck(task interface{}) (interface{}, bool) {
    pe.lock.Lock()
    defer func() {
        // 一开始为 false
        var start bool
        if !pe.guarded {
            // backgroundFlush() 会将 guarded 重新置反
            pe.guarded = true
            start = true
        }
        pe.lock.Unlock()
        // 在第一条 task 加入的时候就会执行 if 中的 backgroundFlush()。后台协程刷 task
        if start {
            pe.backgroundFlush()
        }
    }()
    // 控制 maxTask，>=maxTask 将 container 中 tasks pop, return
    if pe.container.AddTask(task) {
        return pe.container.RemoveAll(), true
    }
    return nil, false
}
```

`addAndCheck()` 中 `AddTask()` 就是在控制最大 `tasks` 数，如果超过就执行 `RemoveAll()` ，将暂存 `container` 的 tasks pop，传递给 `commander` ，后面有 goroutine 循环读取，然后去执行 `tasks`

`backgroundFlush`：

```GO
func (pe *PeriodicalExecutor) backgroundFlush() {
	go func() {
		// flush before quit goroutine to avoid missing tasks
		defer pe.Flush()

		ticker := pe.newTicker(pe.interval)
		defer ticker.Stop()

		var commanded bool
		last := timex.Now()
		for {
			select {
			case vals := <-pe.commander:
				commanded = true
				atomic.AddInt32(&pe.inflight, -1)
				pe.enterExecution()
				pe.confirmChan <- lang.Placeholder
				pe.executeTasks(vals)
				last = timex.Now()
			case <-ticker.Chan():
				if commanded {
					commanded = false
				} else if pe.Flush() {
					last = timex.Now()
				} else if pe.shallQuit(last) {
                    // 特别注意这里，如果检查满足退出条件，则 backgroundFlush 就自行退出
					return
				}
			}
		}
	}()
}
```


`PeriodicalExecutor` 的实现有点意思，它的这个 `backgroundFlush` 任务执行监听 goroutine 不是常驻的，而是过一段时间，会自行检查当前是否存在任务执行的 goroutine（`atomic.LoadInt32(&pe.inflight) == 0`），如果不存在就退出，如下面的代码：

```go
func (pe *PeriodicalExecutor) shallQuit(last time.Duration) (stop bool) {
	if timex.Since(last) <= pe.interval*idleRound {
		return
	}

	// checking pe.inflight and setting pe.guarded should be locked together
	pe.lock.Lock()
	if atomic.LoadInt32(&pe.inflight) == 0 {
        // 如果超过了空闲时间且当前不存在活动 groutine，就退出
		pe.guarded = false
		stop = true
	}
	pe.lock.Unlock()

	return
}
```



```GO
// Flush forces pe to execute tasks.
func (pe *PeriodicalExecutor) Flush() bool {
	pe.enterExecution()
	return pe.executeTasks(func() any {
		pe.lock.Lock()
		defer pe.lock.Unlock()
		return pe.container.RemoveAll()
	}())
}

// Sync lets caller run fn thread-safe with pe, especially for the underlying container.
func (pe *PeriodicalExecutor) Sync(fn func()) {
	pe.lock.Lock()
	defer pe.lock.Unlock()
	fn()
}

// Wait waits the execution to be done.
func (pe *PeriodicalExecutor) Wait() {
	pe.Flush()
	pe.wgBarrier.Guard(func() {
		pe.waitGroup.Wait()
	})
}
```


####    BulkExecutor 实现
如前文描述，`BulkExecutor` 依赖于 `PeriodicalExecutor` 及 `bulkContainer`

```GO
// A BulkExecutor is an executor that can execute tasks on either requirement meets:
	// 1. up to given size of tasks
	// 2. flush interval time elapsed
type BulkExecutor struct {
		executor  *PeriodicalExecutor
		container *bulkContainer
}

func NewBulkExecutor(execute Execute, opts ...BulkOption) *BulkExecutor {
    // 选项模式：在 go-zero 中多处出现。在多配置下，比较好的设计思路
    // https://halls-of-valhalla.org/beta/articles/functional-options-pattern-in-go,54/
    options := newBulkOptions()
    for _, opt := range opts {
        opt(&options)
    }
    // 1. task container: [execute 真正做执行的函数] [maxTasks 执行临界点]
    container := &bulkContainer{
        execute:  execute,
        maxTasks: options.cachedTasks,
    }
    // 2. 可以看出 bulkexecutor 底层依赖 periodicalexecutor
    executor := &BulkExecutor{
        executor:  NewPeriodicalExecutor(options.flushInterval, container),
        container: container,
    }
    return executor
}
```

##  0x03   应用举例
这里引用原文的一个例子，定时同步任务



##  0x04 参考
-   [组件 - executors](https://www.bookstack.cn/read/go-zero-1.1.8-zh/executors.md?wd=%E9%98%BB%E5%A1%9E)
-   [一招让 Kafka 达到最佳吞吐量](https://talkgo.org/t/topic/1945)
-   [core/executors 有数据丢失的风险](https://github.com/zeromicro/go-zero/issues/186)