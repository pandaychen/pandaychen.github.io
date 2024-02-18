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


####   基础定义

1、`TaskContainer`：实例化为 `bulkContainer`、`chunkContainer`，可以理解为一个任务缓冲队列，`AddTask` 用来提示添加任务后是否超过了队列的最大值

```GO
type TaskContainer interface {
		// AddTask adds the task into the container.
		// Returns true if the container needs to be flushed after the addition.
		AddTask(task any) bool
		// Execute handles the collected tasks by the container when flushing.
		Execute(tasks any)
		// RemoveAll removes the contained tasks, and return them.
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


####    PeriodicalExecutor 的实现
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
```

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
                    // 特别注意这里，如果检查满足退出条件，则backgroundFlush就自行退出
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
        // 如果超过了空闲时间且当前不存在活动groutine，就退出
		pe.guarded = false
		stop = true
	}
	pe.lock.Unlock()

	return
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


##  0x04 参考
-   [组件 - executors](https://www.bookstack.cn/read/go-zero-1.1.8-zh/executors.md?wd=%E9%98%BB%E5%A1%9E)
-   [一招让 Kafka 达到最佳吞吐量](https://talkgo.org/t/topic/1945)
-   [core/executors 有数据丢失的风险](https://github.com/zeromicro/go-zero/issues/186)