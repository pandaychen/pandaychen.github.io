---
layout:     post
title:      GoWorkers 通用异步工作队列分析
subtitle:   分析一款基于 Golang 后台队列任务执行框架：jrallison/go-workers
date:       2021-06-30
author:     pandaychen
header-img:
catalog: true
category:   false
tags:
    - 任务队列
    - 队列
    - 异步队列
---


##  0x00    前言
[go-workers](https://github.com/jrallison/go-workers/) 是 sidekiq 的 go 实现，异步队列框架。完全满足了基于 redis queue 的任务调度工作，同时支持了自定义 middleware 供接入者开发延伸需求，仅支持 redis，支持延时任务。作者给出的特点如下：

-   reliable queueing for all queues using brpoplpush
-   handles retries
-   support custom middleware
-   customize concurrency per queue
-   responds to Unix signals to safely wait for jobs to finish before exiting.
-   provides stats on what jobs are currently running
-   well tested


实现一个异步任务队列，通常要考虑如下：

-	Producer，负责把调用者的函数、参数等传入到 Broker 里
-	Consumer，负责从 Broker 里取出消息，并且消费，如果有持久化运行结果的需求，还需要进行持久化
-	选择一个 Producer 和 Consumer 之间序列化和反序列化的协议，通常使用 JSON
-   Consumer 对消费数据的 Acknowledge 机制
-	任务的重试机制、隔离机制（不同属性的任务）、定时任务 / 延时任务 / 普通任务等
-	队列支持的一致性如何？（参考kafka）

##	0x01	系统架构介绍

![go-workers](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/2021/goworkers/go-worker.png)

整体分为下面模块：

1.	[manager](https://github.com/jrallison/go-workers/blob/master/manager.go)：负责 **单个**Queue-name 任务的处理，内含 `1` 个 Fetcher、`N` 个 worker，并发度可配置
    -   [worker](https://github.com/jrallison/go-workers/blob/master/worker.go)：处理具体的任务
    -   [fetcher](https://github.com/jrallison/go-workers/blob/master/fetcher.go)：与 redis 交互，拉取或删除任务
2.	[schedule](https://github.com/jrallison/go-workers/blob/master/scheduled.go)：负责延迟任务发送处理、重试任务处理
3.  [producer](https://github.com/jrallison/go-workers/blob/master/enqueue.go)：任务注册，任务可以设置立即执行或者延迟执行（可以优化为通过 API 接口对外部暴露），项目中是以包方法提供给外部直接调用的方式
	-	普通任务，写入 LIST：https://github.com/jrallison/go-workers/blob/master/enqueue.go#L80
	-	延时任务，写入 ZSET：https://github.com/jrallison/go-workers/blob/master/enqueue.go#L89
4.	[workers](https://github.com/jrallison/go-workers/blob/master/workers.go)：将 schedule 与 manager 模块整合在一起，提供外部启停接口


####	运行流程
以项目提供的 [示例代码](https://github.com/jrallison/go-workers/blob/master/README.md) 为例，从任务执行上看：
-	生产者：将任务放置于指定的 Queue（本质是 LIST，以 queueName 为唯一标识），任务可设置立即执行或延迟执行
-	消费者：由 manager 模块启动多个消费者，消费 queueName 的任务，执行 job 函数

1、前置定义和配置部分 <br>
定义中间件
```golang
type myMiddleware struct{}

func (r *myMiddleware) Call(queue string, message *workers.Msg, next func() bool) (acknowledge bool) {
  // do something before each message is processed
  fmt.Println("before message processing...")
  acknowledge = next()
  fmt.Println("after message processing...")
  // do something after each message is processed
  return
}

//加载中间件
workers.Middleware.Append(&myMiddleware{})
```

初始化包配置：
```golang
workers.Configure(map[string]string{
    // location of redis instance
    "server":  "localhost:6379",
    // instance of the database
    "database":  "0",
    // number of connections to keep open with redis
    "pool":    "30",
    // unique process id for this instance of workers (for proper recovery of inprogress jobs on crash)
    "process": "1",
  })
```

定义worker的处理方法：
```golang
func myJob(message *workers.Msg) {
  // do something with your message
  // message.Jid()
  // message.Args() is a wrapper around go-simplejson (http://godoc.org/github.com/bitly/go-simplejson)
}
```

2、调用package提供的方法，将queueName与处理方法进行绑定，同时设置并发数（可视为不同的业务间隔离，没有注册方法的queueName不应被执行）<br>
```golang
// pull messages from "myqueue" with concurrency of 10
workers.Process("myqueue", myJob, 10)

// pull messages from "myqueue2" with concurrency of 20
workers.Process("myqueue2", myJob, 20)
```

3、任务入队：Enqueue<br>
```golang
// Add a job to a queue
workers.Enqueue("myqueue3", "Add", []int{1, 2})
// Add a job to a queue with retry
workers.EnqueueWithOptions("myqueue3", "Add", []int{1, 2}, workers.EnqueueOptions{Retry: true})
```

若任务需要立即执行，则调用`workers.Enqueue`方法，将任务信息保存到 Redis 的 queueName LIST；若任务需要延迟执行，则需要先将任务保存到共用的Redis 延迟队列（ZSET）中，延时时间设置为score；此外，还可以通过`workers.EnqueueWithOptions`设置任务的属性

![delayqueue](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/delayqueue/delay-queue-Redis.png)

4、启动worker<br>
```golang
// Blocks until process is told to exit via unix signal
workers.Run()
```

5、任务异步处理及获取结果<br>

##  0x02    核心数据结构
1、msg：[消息](https://github.com/jrallison/go-workers/blob/master/msg.go)<br>
```golang
type data struct {
	*simplejson.Json
}

type Msg struct {
	*data
	original string
}
```

##	0x03	实现细节
本小节着重分析下工作流转的核心过程，即manager/fetcher/worker/scheduler等模块的实现。

####	manager 模块
[manager模块](https://github.com/jrallison/go-workers/blob/master/manager.go)的核心逻辑如下：
-	每个queueName都会单独创建一个manager管理协程，负责任务获取（PULL）/调度/分发到worker/完成时acknowledge等处理
-	manager 模块启动时，需要检查是否有残留任务需要处理（即未被ack的任务，如任务处理时异常退出，导致任务未执行完毕，此类任务需重新执行）
-	manager会通过独立goroutine，不停的通过`fetcher.Fetch`[方法](https://github.com/jrallison/go-workers/blob/master/fetcher.go#L56)，将待执行任务从queueName LIST（原始工作任务队列）移动到inprogress LIST（正在执行队列），同时通过channel将任务[异步发送给worker模块](https://github.com/jrallison/go-workers/blob/master/fetcher.go#L107)

1、如何 [ack 任务](https://github.com/jrallison/go-workers/blob/master/fetcher.go#L110)，收到 worker 的处理成功的信息后异步，删除 LIST 中的对应数据
```golang
func (f *fetch) Acknowledge(message *Msg) {
	conn := Config.Pool.Get()
	defer conn.Close()
	conn.Do("lrem", f.inprogressQueue(), -1, message.OriginalJson())
}
```

####	worker 模块
worker模块的核心逻辑如下：
1.	从与manager共享的任务channel中[获取任务并执行](https://github.com/jrallison/go-workers/blob/master/worker.go#L28)
2.	通过middleware中间件形式将处理状态返回，见下面的`process`方法
3.	若任务处理成功，则通过channel通知manager模块，将任务从inprogress队列删除，即是acknowledge方式（可借鉴）
4.	若任务失败需要重试，则将重试信息放入retry队列（重试间隔时间成指数级递增）

下面列举下worker核心的方法：

1、执行任务的 [逻辑](https://github.com/jrallison/go-workers/blob/master/worker.go#L57)<br>
```golang
func (w *worker) process(message *Msg) (acknowledge bool) {
	acknowledge = true

	defer func() {
		recover()
	}()

	//以中间件形式运行并返回
	return w.manager.mids.call(w.manager.queueName(), message, func() {
		w.manager.job(message)
	})
}
```

####	schedule 模块
scheduler模块主要负责对延迟任务和重试任务的[处理](https://github.com/jrallison/go-workers/blob/master/manager.go#L86)，这里巧妙的利用了延时任务和重试任务本质上其实是相同的（都是等待一段时间后出发的特点）
-	延迟任务：ZSET 中 `score` 为任务执行时间戳，利用 `zrangebyscore`指令，`score` 为 `-inf -> now`，获取到期的可执行任务，然后将任务从 ZSET 中删除，并放入对应 queueName 队列
-	重试任务：同上

####	fetcher 模块
[fetcher模块](https://github.com/jrallison/go-workers/blob/master/fetcher.go)主要封装了大部分操作Redis LIST任务队列的方法，**从设计上说，使得逻辑层与数据层分离，仅通过 fetcher 模块与 redis 交互，其他模块仅需要调用fetcher提供的方法即可**：
-	`Acknowledge`方法：ack 后删除任务
-	`processOldMessages`方法：获取inprocessing队列中未被ack的方法
-	`Fetch`方法：从Redis相关队列获取任务



####	hooks实现
[hook](https://github.com/jrallison/go-workers/blob/master/hooks.go)


##	0x04	消费者一致性语义
参考另外一篇[文章](https://pandaychen.github.io/2022/01/01/A-KAFKA-USAGE-SUMUP-2/#%E6%B6%88%E8%B4%B9%E8%80%85%E8%AF%AD%E4%B9%89)，可以简单的嵌套一下场景：

-	schedule 模块从 ZSET 中检索可执行任务，然后先调用 `ZREM`，再调用 `LPUSH` 将任务放入 queueName 中，如果在 `ZREM` 之后，`LPUSH` 之前挂掉，则任务丢失且无感知，这里是 at most once模型
-	manager 模块将任务下发后，worker 处理完毕，在 ack 之前进程挂掉，下次再次启动 manager 会进行任务重发。即ack之前的异常退出都被视作任务处理失败，触发重入，这里是 at least once模型（PS：在这种情况下，开发者需要自行实现消费者的幂等性）

##	0x05	总结
本项目基于 golang 实现了通用的任务系统（相对于 machinery 较为轻量级），将具体业务逻辑剥离，保留了整个最核心的任务拆分、任务调度、任务作业等功能，局限是仅支持 Redis 作为 Broker，此外，go-workers提供的方法均为包类型，直接调用即可。

从功能上说，该系统提供了业务隔离设计、延迟任务支持，失败重试能力以及ack任务的功能。此外，提供了middleware的中间件实现，将日志、状态统计、错误重试、用户自定义钩子等嵌入到任务处理流程中，可以借鉴

##  0x06    参考
-   [Sidekiq compatible background workers in golang](https://github.com/jrallison/go-workers)
-   [[12 台减至 3 台] 用 Golang 重写 Sidekiq 的 worker](https://ruby-china.org/topics/26785)
-	[golang任务作业系统实现分析](http://www.xiaocc.xyz/2020-04-19/golang%E4%BB%BB%E5%8A%A1%E4%BD%9C%E4%B8%9A%E7%B3%BB%E7%BB%9F/)

转载请注明出处，本文采用 [CC4.0](http://creativecommons.org/licenses/by-nc-nd/4.0/) 协议授权