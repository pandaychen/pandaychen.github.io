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
    commander chan any
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


##  0x03   应用举例


##  0x04 参考
-   [组件 - executors](https://www.bookstack.cn/read/go-zero-1.1.8-zh/executors.md?wd=%E9%98%BB%E5%A1%9E)
-   [一招让 Kafka 达到最佳吞吐量](https://talkgo.org/t/topic/1945)
-   []()