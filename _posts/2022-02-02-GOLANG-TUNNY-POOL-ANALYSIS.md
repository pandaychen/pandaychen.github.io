---
layout: post
title: Golang 并发协程池实现分析（四）
subtitle: 分析 Jeffail/tunny 协程池实现
date: 2022-02-02
author: pandaychen
header-img: img/golang-horse-fly.png
catalog: true
category: false
tags:
  - Golang
  - 协程池
---

## 0x00 前言
本文分析下[tunny](https://github.com/Jeffail/tunny)协程池的实现。先说结论：tunny的并发控制核心是固定数量的工作goroutine，监听唯一的任务管道，即全局`reqChan`，每个工作goroutine都需要经历**等待获取任务->获取任务->获取任务参数->执行任务->异步返回任务结果->再次等待获取任务**这一完整的过程，从而Pool通过此实现了控制并发。

由于工作协程goroutine的数量是固定的，所以tunny的并发数也是固定的

## 0x01 tunny 协程池使用
要点；
-	全局池，初始化处理方法
-	在逻辑较重的地方，使用`pool.Process(input)`处理并返回
-	如果项目针对多个方法都需要使用协程池，那么有两个方法
	1.	每个方法都开启独立的协程池
	2.	仅开启`1`个协程池，根据传入参数做`switch`

```golang
import (
	"io/ioutil"
	"net/http"
	"runtime"

	"github.com/Jeffail/tunny"
)

func main() {
	numCPUs := runtime.NumCPU()

	pool := tunny.NewFunc(numCPUs, func(payload interface{}) interface{} {
		var result []byte
		// TODO: Something CPU heavy with payload	
		return result
	})
	defer pool.Close()

	http.HandleFunc("/work", func(w http.ResponseWriter, r *http.Request) {
		input, err := ioutil.ReadAll(r.Body)
		if err != nil {
			http.Error(w, "Internal error", http.StatusInternalServerError)
		}
		defer r.Body.Close()

		// Funnel this work into our pool. This call is synchronous and will
		// block until the job is completed.
		result := pool.Process(input)

		w.Write(result.([]byte))
	})

	http.ListenAndServe(":8080", nil)
}
```

##	数据结构
1、Pool<br>
注意`reqChan`这个成员，初始化长度为`1`：`reqChan: make(chan workRequest)`，各个空闲的worker把自己的`workRequest.jobChan`，`workRequest.retChan`发送给`Pool`，`Pool`会把待处理的`payload`放入`workRequest.jobChan`，并且等待从`workRequest.retChan`获取执行结果，[代码在此](https://github.com/Jeffail/tunny/blob/master/tunny.go#L153)

```golang
// Pool is a struct that manages a collection of workers, each with their own
// goroutine. The Pool can initialize, expand, compress and close the workers,
// as well as processing jobs with the workers synchronously.
type Pool struct {
	queuedJobs int64

	ctor    func() Worker
	workers []*workerWrapper
	reqChan chan workRequest	//有点像令牌

	workerMut sync.Mutex
}
```

2、Worker<br>
```GO
type Worker interface {
	// Process will synchronously perform a job and return the result.
	Process(interface{}) interface{}

	// BlockUntilReady is called before each job is processed and must block the
	// calling goroutine until the Worker is ready to process the next job.
	BlockUntilReady()

	// Interrupt is called when a job is cancelled. The worker is responsible
	// for unblocking the Process implementation.
	Interrupt()

	// Terminate is called when a Worker is removed from the processing pool
	// and is responsible for cleaning up any held resources.
	Terminate()
}
```

3、workerWrapper<br>

4、workRequest<br>
`workRequest`可以视为一个请求处理单元


##	0x02	分析
tunny的核心原理如下图，涵盖了tunny从资源池中获取goroutine并进行处理的逻辑：
![tunny-principle]()

先梳理下tunny的要点，后续再详细分析：

1.	tunny将goroutine处理单元封装为`workWrapper`并且固定数量，由此可以对goroutine的数目进行限制
2.	`workerWrapper.run()`作为一个goroutine，内部负责具体事务的处理；当一个worker空闲时，会将一个`workRequest`（可看作是请求处理单元）放入`reqChan`，并阻塞等待使用方的调用，核心代码[在此](https://github.com/Jeffail/tunny/blob/master/worker.go#L94)
3.	`workRequest`包含两个channel，其中`jobChan`用于传入调用方的输入，`retChan`用于给调用方返回执行结果
4.	调用方会从pool的`reqChan`中获取一个`workRequest`请求处理单元，并在`workRequest.jobChan`中传参，这样`workerWrapper.run()`中就会继续进行`work.process`执行；处理结束之后将结果通过`workRequest.retChan`返回给调用方，然后继续通过`reqChan <- workRequest`阻塞等待下一个调用方的处理
5.	`workerWrapper.run()`中的work是一个需要用户实现的接口，必须实现`Process(interface{}) interface{}`，即业务逻辑方法！


##  0x03  生产者
tunny提供了`3`种方法：[ProcessX](https://github.com/Jeffail/tunny/blob/master/tunny.go#L153)，用来插入任务列表

-	`p.Process()`：会一直阻塞直到任务完成，即使当前没有空闲 worker 也会阻塞
-	`p.ProcessTimed()`：带超时的`Process()`，支持传入一个超时时间，如果超过这个时间还没有空闲 worker，或者任务还没有处理完成，就会终止，并返回错误
-	`p.ProcessCtx()`：当前 context 状态变为`Done`之后，任务也会停止执行。context 会由于超时、取消等原因切换为`Done`状态。

通常，在协程池中超时有 `2` 种情况：
1.	等不到空闲的 worker：所有 worker 一直处理繁忙状态，正在处理的任务比较耗时，无法短时间内完成
2.	任务本身比较耗时，必须加一个超时时间方式任务执行被长期阻塞

```GOLANG
// Process will use the Pool to process a payload and synchronously return the
// result. Process can be called safely by any goroutines, but will panic if the
// Pool has been stopped.
func (p *Pool) Process(payload interface{}) interface{} {
	atomic.AddInt64(&p.queuedJobs, 1)

  //1. 获取一个执行令牌
	request, open := <-p.reqChan
	if !open {
		panic(ErrPoolNotRunning)
	}

  //2. 将运行参数通过channel传递给执行goroutine
	request.jobChan <- payload

  //3. 阻塞等待获取结果
	payload, open = <-request.retChan
	if !open {
		panic(ErrWorkerClosed)
	}

  //4. 返回调用方结果
	atomic.AddInt64(&p.queuedJobs, -1)
	return payload
}

// ProcessTimed will use the Pool to process a payload and synchronously return
// the result. If the timeout occurs before the job has finished the worker will
// be interrupted and ErrJobTimedOut will be returned. ProcessTimed can be
// called safely by any goroutines.
func (p *Pool) ProcessTimed(
	payload interface{},
	timeout time.Duration,
) (interface{}, error) {
	atomic.AddInt64(&p.queuedJobs, 1)
	defer atomic.AddInt64(&p.queuedJobs, -1)

	tout := time.NewTimer(timeout)

	var request workRequest
	var open bool

	select {
	case request, open = <-p.reqChan:
		if !open {
			return nil, ErrPoolNotRunning
		}
	case <-tout.C:
		return nil, ErrJobTimedOut
	}

	select {
	case request.jobChan <- payload:
	case <-tout.C:
		request.interruptFunc()
		return nil, ErrJobTimedOut
	}

	select {
	case payload, open = <-request.retChan:
		if !open {
			return nil, ErrWorkerClosed
		}
	case <-tout.C:
		request.interruptFunc()
		return nil, ErrJobTimedOut
	}

	tout.Stop()
	return payload, nil
}

// ProcessCtx will use the Pool to process a payload and synchronously return
// the result. If the context cancels before the job has finished the worker will
// be interrupted and ErrJobTimedOut will be returned. ProcessCtx can be
// called safely by any goroutines.
func (p *Pool) ProcessCtx(ctx context.Context, payload interface{}) (interface{}, error) {
	atomic.AddInt64(&p.queuedJobs, 1)
	defer atomic.AddInt64(&p.queuedJobs, -1)

	var request workRequest
	var open bool

	select {
	case request, open = <-p.reqChan:
		if !open {
			return nil, ErrPoolNotRunning
		}
	case <-ctx.Done():
		return nil, ctx.Err()
	}

	select {
	case request.jobChan <- payload:
	case <-ctx.Done():
		request.interruptFunc()
		return nil, ctx.Err()
	}

	select {
	case payload, open = <-request.retChan:
		if !open {
			return nil, ErrWorkerClosed
		}
	case <-ctx.Done():
		request.interruptFunc()
		return nil, ctx.Err()
	}

	return payload, nil
}
```

##  0x04  任务传输


##  0x05  消费者：任务执行
worker的代码[实现](https://github.com/Jeffail/tunny/blob/master/worker.go)

```GOLANG
func (w *workerWrapper) run() {
	jobChan, retChan := make(chan interface{}), make(chan interface{})
	defer func() {
		w.worker.Terminate()
		close(retChan)
		close(w.closedChan)
	}()

	for {
		// NOTE: Blocking here will prevent the worker from closing down.
		w.worker.BlockUntilReady()
		select {
		case w.reqChan <- workRequest{
			jobChan:       jobChan,
			retChan:       retChan,
			interruptFunc: w.interrupt,
		}:
			select {
			case payload := <-jobChan:
				result := w.worker.Process(payload)
				select {
				case retChan <- result:
				case <-w.interruptChan:
					w.interruptChan = make(chan struct{})
				}
			case <-w.interruptChan:
				w.interruptChan = make(chan struct{})
			}
		case <-w.closeChan:
			return
		}
	}
}
```

##  0x06  结果通知

##  0x07  一些细节

##  0x08  总结
tunny的思路与ants有较大的区别：
-	tunny仅支持同步的方式执行任务（适用于需要同步等待结果的场景），虽然任务在另一个 goroutine 执行，但是提交任务的 goroutine 必须等待结果返回或超时，不能做其他事情
-	tunny 为了支持超时和取消，设计了多个channel用于和执行任务的 goroutine 通信。一次任务执行的过程涉及多次通信，性能会有损失
-	而ants完全是异步的任务执行流程，相比tunny性能是稍高一些的。但是也因为它的异步特性，导致没有任务超时、取消这些机制。而且如果需要收集结果，必须要自己编写额外的代码

## 0x09 参考
- [Go 每日一库之 tunny](https://darjun.github.io/2021/06/10/godailylib/tunny/)
- [分析一个简单的goroutine资源池](https://www.cnblogs.com/charlieroro/p/15735779.html)