---
layout: post
title: Golang 的分布式任务队列：machinery 分析（一）
subtitle: 如何使用 Golang 实现通用的任务调度作业模型
date: 2020-11-03
author: pandaychen
catalog: true
tags:
    - Machinery
    - 队列
---

## 0x00 前言
Machinery 是一个基于分布式消息分发的异步任务队列框架，有点类似于 Celery。异步任务的主要作用是解耦，即将需要长时间执行的逻辑单独处理，避免响应延时带来的问题；此外借助于 machinery，可以完成各种复杂类型任务的调度、编排、结果记录等功能，完美的支持了日常业务场景。考虑如下几个问题：

-  Machinery 的架构 / 模块划分的实现
-  Machinery 如何有无 HA 实现的机制？
-  Machinery 的 worker 有无负载均衡的机制？

![task-queue](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/2022/machinery/overview.png)

####  基础使用例子


####  特点
-  任务重试机制
-  延迟（延时）任务支持
-  任务回调机制
-  任务结果记录
-  支持 Workflow 模式：Chain/Group/Chord
-  多 Brokers 支持：Redis/AMQP/... 等
-  多 Backends 支持：Redis/Memcache/AMQP/MongoDB/... 等

此外，在代码上，Machinery 的实现非常的优雅，非常值得阅读。

####  子模块介绍
对于异步队列，一般离不开经典的 ** 生产者 - 消费者 ** 模型，由生产者（`Server`）生成任务并放进队列（`Brokers`）中，由消费者（`Worker`）从队列中领取任务并执行（订阅 && 执行），执行结果存储（`Backend`）。Machinery 框架由以下组件（支持多机分布式部署）：

![workflow](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/2022/machinery/machinery3.png)

-  Server 模块：处理客户端同步请求，并生成异步任务，放置进任务队列（任务的发布方，负责将用户定义的任务发布出去及获取任务状态）
-  Worker 模块：工作线程，从任务队列中消费任务并执行
-  Broker 模块：任务队列，存储序列化后的任务
-  Backend 模块：后端存储，存储任务执行状态和结果
-  Signature 模块：任务实体，用来描述任务的执行过程、所需参数等信息

1、Server<br>
一般 Server 的逻辑会嵌入到对外部暴露的服务（如 Apiserver）实现，本质上像一个轻客户端；Server 是消息的生产者，通常嵌入到业务模块中，由业务方调用，负责生产、发布异步任务

2、Worker<br>
Worker 是消费者、工作模块，用于监听、消费 / 执行异步任务；如果任务执行失败，Worker 会调用 Server 的方法重新发布任务，直至任务重试次数归零

3、Broker<br>
Broker 是一个消息队列中间件，提供任务的持久化（暂时保存产生的任务以便于消费   ）。Server 和 Worker 通过 Broker 进行通信，在一定程度上解耦两个子模块

4、Backend<br>
Backend 是后端存储，保存了任务的执行状态和最终执行结果

5、Signature / Task<br>
任务：自定义函数，用来描述任务的执行过程。函数名即为任务名称，函数传参即为任务参数，函数返回结果即为任务输出。

6、State<br>
即任务状态：如 `PENDING`、`SUCCESS`、`FAILURE` 等。任务状态及结果会被设置 TTL（默认为 `1 hour`），也即每个任务的状态最多保持 `1` 个小时。

####  基础的工作流程
![work-flow](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/2022/machinery/machinery-work-flow1.png)

Machinery 基本的工作流程如下：
1. 由 Server 生成并发布任务，推送到 Broker 中
2. Worker 通过 Key 向 Broker 订阅任务，当 Key 相同的任务到达时，Worker 消费任务
3. Worker 执行任务
4. Worker 将执行结果（终态：`SUCCESS`、`FAILURE`）存储至 Backend 模块

下面，本文基于 [V1](https://github.com/RichardKnop/machinery/tree/master/v1) 版本对代码进行完整的分析。

## 0x01  核心代码分析

####  任务状态机：状态维护及变迁
任务作为一个框架的原子单位，流经了系统的各个子模块，其生命周期起于发布，终于结果（任务有且只有成功和失败两个终态）。所以对其建立 [状态流转模型](https://github.com/RichardKnop/machinery/blob/master/v1/tasks/state.go) 是很有用的，在 Machinery 中每个任务状态有 `PENDING`、`RECEIVED`、`STARTED`、`RETRY`、`SUCCESS`/`FAILURE` 几种，任务在生成和处理的不同阶段的状态机如下图所示：
-  初始化时，Server 插入任务时状态为 `PENDING`
-  Worker 收到任务时状态为 `RECEIVED`
-  Worker 开始执行任务将状态修改为 `STARTED`
-  当设置 `retry` 标志时，任务一旦失败，状态被修改为 `RETRY`，等待后续轮转执行
-  任务最终的执行结果为 `SUCCESS` 或 `FAILURE`，`FAILURE` 时伴随失败原因存储至 Backend

![state](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/2022/machinery/machinery-work-state1.png)

由上图可知，除 `PENDING` 状态外，任务的大部分生命周期都位于 Worker 内部流转。当任务在 worker 内部进行状态流转时，Worker 会将状态和结果信息上报 Backend。

生产者 Server 发布任务 -> Broker -> 消费者竞争一个任务，然后进行消费 -> (可选：** 消费后向 Broker 确认已经消费，然后 Broker 删除此任务，否则将超时重发任务 **) -> Backend 保存结果

1、`PENDING` 状态 <br>
一旦任务被发布到消息队列中，它的初始化状态就是 `PENDING`

2、`RECEIVED` 状态 <br>
当 Worker 从消息队列中读取任务，会将任务标记为 `RECEIVED`，表示已有 worker 在处理该任务。此时消息队列中的 Task 被标记删除，不能再被读取

3、`STARTED`<br>
`STARTED` 状态标识正在被执行的任务

4、`RETRY`<br>
当任务发生异常或主动请求重试时，Worker 会将任务标记为 `RETRY` 状态，开启重试机制

5、`SUCCESS`/`FAILURE`<br>
任务的最终结果状态

## 0x01 结构 && 通用方法
本文以 Redis 为中间件作为分析。

####  Signature
`Signature` 即任务实体，用来描述任务的执行过程、所需参数等信息。
```golang
// Signature represents a single task invocation
type Signature struct {
	UUID           string
	Name           string
	RoutingKey     string
	ETA            *time.Time
	GroupUUID      string
	GroupTaskCount int
	Args           []Arg
	Headers        Headers
	Priority       uint8
	Immutable      bool
	RetryCount     int
	RetryTimeout   int
	OnSuccess      []*Signature
	OnError        []*Signature
	ChordCallback  *Signature
	//MessageGroupId for Broker, e.g. SQS
	BrokerMessageGroupId string
	//ReceiptHandle of SQS Message
	SQSReceiptHandle string
	// StopTaskDeletionOnError used with sqs when we want to send failed messages to dlq,
	// and don't want machinery to delete from source queue
	StopTaskDeletionOnError bool
	// IgnoreWhenTaskNotRegistered auto removes the request when there is no handeler available
	// When this is true a task with no handler will be ignored and not placed back in the queue
	IgnoreWhenTaskNotRegistered bool
}
```

####  任务
[](https://github.com/RichardKnop/machinery/blob/master/v1/tasks/task.go#L22)
```golang
// Task wraps a signature and methods used to reflect task arguments and
// return values after invoking the task
type Task struct {
	TaskFunc   reflect.Value
	UseContext bool
	Context    context.Context
	Args       []reflect.Value
}
```

`Signature` 与 `Task` 的区别是：TODO


####  Server 模块
[实现](https://github.com/RichardKnop/machinery/blob/master/v1/server.go)
```golang
type Server struct {
	config            *config.Config
	registeredTasks   *sync.Map
	broker            brokersiface.Broker
	backend           backendsiface.Backend
	lock              lockiface.Lock
	scheduler         *cron.Cron
	prePublishHandler func(*tasks.Signature)
}
```

####  Broker 模块
Broker 的 [通用定义](https://github.com/RichardKnop/machinery/blob/master/v1/brokers/iface/interfaces.go) 如下，支持的类型必须实现下面的方法列表：
```golang
// Broker - a common interface for all brokers
type Broker interface {
	GetConfig() *config.Config
	SetRegisteredTaskNames(names []string)
	IsTaskRegistered(name string) bool
	StartConsuming(consumerTag string, concurrency int, p TaskProcessor) (bool, error)
	StopConsuming()
	Publish(ctx context.Context, task *tasks.Signature) error
	GetPendingTasks(queue string) ([]*tasks.Signature, error)
	GetDelayedTasks() ([]*tasks.Signature, error)
	AdjustRoutingKey(s *tasks.Signature)
}

// TaskProcessor - can process a delivered task
// This will probably always be a worker instance
type TaskProcessor interface {
	Process(signature *tasks.Signature) error
	CustomQueue() string
	PreConsumeHandler() bool
}
```

####  Worker 模块
Worker 模块的 [定义](https://github.com/RichardKnop/machinery/blob/master/v1/worker.go#L24) 如下：
```golang
// Worker represents a single worker process
type Worker struct {
	server            *Server
	ConsumerTag       string
	Concurrency       int
	Queue             string
	errorHandler      func(err error)
	preTaskHandler    func(*tasks.Signature)
	postTaskHandler   func(*tasks.Signature)
	preConsumeHandler func(*Worker) bool
}
```

####  Backend 模块
Backend 的 [定义](https://github.com/RichardKnop/machinery/blob/master/v1/backends/iface/interfaces.go) 如下，Backend 接口提供了状态更新函数，每次任务发生状态转移时，Worker 会调用 `SetState*` 系列方法，向 Backend 上报状态。
```golang
// Backend - a common interface for all result backends
type Backend interface {
	// Group related functions
	InitGroup(groupUUID string, taskUUIDs []string) error
	GroupCompleted(groupUUID string, groupTaskCount int) (bool, error)
	GroupTaskStates(groupUUID string, groupTaskCount int) ([]*tasks.TaskState, error)
	TriggerChord(groupUUID string) (bool, error)

	// Setting / getting task state
	SetStatePending(signature *tasks.Signature) error
	SetStateReceived(signature *tasks.Signature) error
	SetStateStarted(signature *tasks.Signature) error
	SetStateRetry(signature *tasks.Signature) error
	SetStateSuccess(signature *tasks.Signature, results []*tasks.TaskResult) error
	SetStateFailure(signature *tasks.Signature, err string) error
	GetState(taskUUID string) (*tasks.TaskState, error)

	// Purging stored stored tasks states and group meta data
	IsAMQP() bool
	PurgeState(taskUUID string) error
	PurgeGroupMeta(groupUUID string) error
}
```

## 0x02  业务逻辑分析 - Broker

## 0x03  业务逻辑分析 - Worker

## 0x04  业务逻辑分析 - 任务视角
Machinery 的最大的特点在于它利用消息队列的能力实现了各种模式的任务投递，如单任务、批任务、工作流任务和链式任务，这些模式底层都是调用 `SendTaskWithContext` 方法来完成任务投递。本小节以任务维度来分析

####  任务的一般流程
任务是 Machinery 中的最基础的元素。从代码层面来看，一个任务就是一个执行函数。一个任务的典型的处理流程如下：
1. 任务创建（Server）
2. 任务注册
3. 任务发布
4. 任务执行
5. 结果获取


####  任务结构
一个任务称作为 `Signature`，其结构如下：
```golang
// Signature represents a single task invocation
type Signature struct {
	UUID           string
	Name           string
	RoutingKey     string
	ETA            *time.Time
	GroupUUID      string
	GroupTaskCount int
	Args           []Arg
	Headers        Headers
	Priority       uint8
	Immutable      bool
	RetryCount     int
	RetryTimeout   int
	OnSuccess      []*Signature
	OnError        []*Signature
	ChordCallback  *Signature
	//MessageGroupId for Broker, e.g. SQS
	BrokerMessageGroupId string
	//ReceiptHandle of SQS Message
	SQSReceiptHandle string
	// StopTaskDeletionOnError used with sqs when we want to send failed messages to dlq,
	// and don't want machinery to delete from source queue
	StopTaskDeletionOnError bool
	// IgnoreWhenTaskNotRegistered auto removes the request when there is no handeler available
	// When this is true a task with no handler will be ignored and not placed back in the queue
	IgnoreWhenTaskNotRegistered bool
}
```

####  任务模式
Machinery 提供了丰富的任务模式，包含有单任务、批任务、工作流任务和链式任务等四种。Machinery 通过任务回调字段与任务投递相结合的方式实现了这几种模式，对比如下：


| 模式 | 任务投递的方法 | 介绍 | Server 行为 && 说明 |Worker 行为 && 说明 |
| :-----:| :----: | :----: | :----: |:----: |
| 单任务模式  | SendTaskWithContext | 单个原子任务 |Server 每次只会投递一个任务，投递完成后函数返回 | Worker 每次接收一个任务并处理 |
| 批任务 | SendGroupWithContext | 由多个单任务组成一个 Group | Server 会并发投递 Group 内的任务，直到所有任务投递完成以后函数返回 | Worker 行为与单任务 Worker 一致 |
| 工作流任务 | SendChordWithContext| 由多个单任务组成一个 Group，再增加一个回调函数成为 Chord 任务 |Server 行为与批任务模式一致 | Worker 在所有任务都成功以后（通过 Backend 状态判断）会将所有任务的输出结果作为回调函数的参数，重新发起一次回调函数的任务投递 |
| 链式任务  | SendChainWithContext | 由多个任务按序组成一个 Chain，通过 `OnSuccess` 回调字段链接任务序列 |Server 行为与单任务模式一致 | Worker 在任务执行成功以后投递 `OnSuccess` 指定的下一个任务，直至所有任务完成投递 |


####  任务投递（分发到 Broker）
任务的发布通过 Broker 的 `Publish` 方法 [实现](https://github.com/RichardKnop/machinery/blob/master/v1/brokers/redis/redis.go#L191)，接口中对延时（定时）任务和实时任务分别作了处理（通过 `signature.ETA` 字段区分实时和延时任务）；Machinery 使用 Redis List（RPUSH）来保存实时任务，而对于延时任务，则使用带权重的有序集（ZSet）保存任务。

-  实时任务：直接发布到执行队列，对应的路由键是 `signature.RoutingKey`
-  延时任务：使用任务执行的时间戳做权重，发布到延时队列，对应的路由键是 `Broker.redisDelayedTasksKey`（基于 Redis 的 ZSet 机制实现，原理见 [Redis 应用梳理篇（一）](https://pandaychen.github.io/2020/01/21/A-REDIS-USAGE-SUMUP/# 延迟队列)）

```golang
// Publish places a new message on the default queue
func (b *Broker) Publish(ctx context.Context, signature *tasks.Signature) error {
  b.Broker.AdjustRoutingKey(signature)

  // 任务序列化
  msg, err := json.Marshal(signature)
  if err != nil {
    return fmt.Errorf("JSON marshal error: %s", err)
  }

  conn := b.open()
  defer conn.Close()

  // 延时任务的设置
  if signature.ETA != nil {
    now := time.Now().UTC()

    if signature.ETA.After(now) {
      score := signature.ETA.UnixNano()
      // 延时任务，使用时间戳作为权重
      _, err = conn.Do("ZADD", redisDelayedTasksKey, score, msg)
      return err
    }
  }

  // 添加到消息队列
  _, err = conn.Do("RPUSH", signature.RoutingKey, msg)
  return err
}
```

![img-task-queue-mode]()

1、实时任务 <br>
通过 Redis 的 `RPUSH` 和 `LPOP` 原语实现，执行队列的任务会立刻被 Worker 消费

2、延时任务队列 <br>
延时队列则使用带权重的有序集（Sorted set）保存任务，这样 Worker 每次只需要处理 score 最小的任务，减少搜索时间。这里看下延时任务是如何投递的。在每个 Worker 节点启动时，Machinery 会启动一个协程，用于监听延时队列，将到达执行时间的任务投递到执行队列。[调用路径](https://github.com/RichardKnop/machinery/blob/master/v1/worker.go#L78) 是 `worker.LaunchAsync`-->`broker.StartConsuming`。在协程中，使用 `for...select` 多路复用机制接收退出信号，同时在每个周期，都会通过 `backend.nextDelayedTask` 方法拉取需要被执行的延时任务，核心代码如下：

```golang
// StartConsuming enters a loop and waits for incoming messages
func (b *Broker) StartConsuming(consumerTag string, concurrency int, taskProcessor iface.TaskProcessor) (bool, error) {
	b.consumingWG.Add(1)
	defer b.consumingWG.Done()

	if concurrency < 1 {
		concurrency = runtime.NumCPU() * 2
	}

	b.Broker.StartConsuming(consumerTag, concurrency, taskProcessor)

	conn := b.open()
	defer conn.Close()

	// Ping the server to make sure connection is live
	_, err := conn.Do("PING")
	if err != nil {
		b.GetRetryFunc()(b.GetRetryStopChan())

		// Return err if retry is still true.
		// If retry is false, broker.StopConsuming() has been called and
		// therefore Redis might have been stopped. Return nil exit
		// StartConsuming()
		if b.GetRetry() {
			return b.GetRetry(), err
		}
		return b.GetRetry(), errs.ErrConsumerStopped
	}

	// Channel to which we will push tasks ready for processing by worker
	deliveries := make(chan []byte, concurrency)
	pool := make(chan struct{}, concurrency)

	// initialize worker pool with maxWorkers workers
	for i := 0; i < concurrency; i++ {
		pool <- struct{}{}
	}

	// A receiving goroutine keeps popping messages from the queue by BLPOP
	// If the message is valid and can be unmarshaled into a proper structure
	// we send it to the deliveries channel

   // 处理普通实时任务
	go func() {

		log.INFO.Print("[*] Waiting for messages. To exit press CTRL+C")

		for {
			select {
			// A way to stop this goroutine from b.StopConsuming
			case <-b.GetStopChan():
				close(deliveries)
				return
			case <-pool:
				select {
				case <-b.GetStopChan():
					close(deliveries)
					return
				default:
				}

				if taskProcessor.PreConsumeHandler() {
					task, _ := b.nextTask(getQueue(b.GetConfig(), taskProcessor))
					//TODO: should this error be ignored?
					if len(task) > 0 {
						deliveries <- task
					}
				}

				pool <- struct{}{}
			}
		}
	}()

	// A goroutine to watch for delayed tasks and push them to deliveries
	// channel for consumption by the worker
	b.delayedWG.Add(1)
   // 处理延时任务
	go func() {
		defer b.delayedWG.Done()

		for {
			select {
			// A way to stop this goroutine from b.StopConsuming
			case <-b.GetStopChan():
				return
			default:
				task, err := b.nextDelayedTask(b.redisDelayedTasksKey)
				if err != nil {
					continue
				}

				signature := new(tasks.Signature)
				decoder := json.NewDecoder(bytes.NewReader(task))
				decoder.UseNumber()
				if err := decoder.Decode(signature); err != nil {
					log.ERROR.Print(errs.NewErrCouldNotUnmarshalTaskSignature(task, err))
				}

            // 将任务发布到执行队列
				if err := b.Publish(context.Background(), signature); err != nil {
					log.ERROR.Print(err)
				}
			}
		}
	}()

	if err := b.consume(deliveries, concurrency, taskProcessor); err != nil {
		return b.GetRetry(), err
	}

	// Waiting for any tasks being processed to finish
	b.processingWG.Wait()

	return b.GetRetry(), nil
}
```

这里再看下延时任务是如何被取出来的：
```golang
// nextDelayedTask pops a value from the ZSET key using WATCH/MULTI/EXEC commands.
// https://github.com/gomodule/redigo/blob/master/redis/zpop_example_test.go
func (b *Broker) nextDelayedTask(key string) (result []byte, err error) {
	conn := b.open()
	defer conn.Close()

	defer func() {
		// Return connection to normal state on error.
		// https://redis.io/commands/discard
		// https://redis.io/commands/unwatch
		if err == redis.ErrNil {
			conn.Do("UNWATCH")
		} else if err != nil {
			conn.Do("DISCARD")
		}
	}()

	var (
		items [][]byte
		reply interface{}
	)

	pollPeriod := 500 // default poll period for delayed tasks
	if b.GetConfig().Redis != nil {
		configuredPollPeriod := b.GetConfig().Redis.DelayedTasksPollPeriod
		// the default period is 0, which bombards redis with requests, despite
		// our intention of doing the opposite
		if configuredPollPeriod > 0 {
			pollPeriod = configuredPollPeriod
		}
	}

	for {
		// Space out queries to ZSET so we don't bombard redis
		// server with relentless ZRANGEBYSCOREs
		time.Sleep(time.Duration(pollPeriod) * time.Millisecond)
		if _, err = conn.Do("WATCH", key); err != nil {
			return
		}

		now := time.Now().UTC().UnixNano()

		// https://redis.io/commands/zrangebyscore
		items, err = redis.ByteSlices(conn.Do("ZRANGEBYSCORE",key,0,now,"LIMIT",0,1,))
		if err != nil {
			return
		}
		if len(items) != 1 {
			err = redis.ErrNil
			return
		}

		_ = conn.Send("MULTI")
		_ = conn.Send("ZREM", key, items[0])
		reply, err = conn.Do("EXEC")
		if err != nil {
			return
		}

		if reply != nil {
			result = items[0]
			break
		}
	}

	return
}
```

一个容易被忽略的细节：任务的一致性。即任务从延时队列投递到执行队列的过程中，需要流经 Broker。在分布式场景下，多个 Broker 同时进行消息转发，势必会带来数据一致性问题。所以在上面的代码中可以看到基于 Redis 的事务操作机制，保证了延时任务的分布式一致性。（思考下为何实时任务不需要使用事务？）

####  任务的消费
Machinery 的任务消费有下面几个细节（又是一个基于 channel 的生产 - 消费模型）：
1. 使用 buffered channel 机制来实现并发（较简单，消费令牌 / 归还令牌的机制），单个 goroutine 从 Broker 拉取实时任务，多个 goroutine 并发消费处理任务
2. 拉取任务的 goroutine 和执行任务的 goroutine 之间，通过 `deliveries` 异步化处理，`pool` 起到了令牌池的作用，用于并发控制
3. 可以通过调大 `pool` 的数量来提高并发度（即增大执行任务的 goroutine 数量）


```golang
func (b *Broker) StartConsuming(consumerTag string, concurrency int, taskProcessor iface.TaskProcessor) (bool, error) {
   //...
	deliveries := make(chan []byte, concurrency)    // 模拟任务队列
	pool := make(chan struct{}, concurrency)        // 信号量池

	for i := 0; i < concurrency; i++ {
		pool <- struct{}{}
	}

	// 单个 goroutine：获取消息，将消息从 broker 转发到任务队列
	go func() {
		for {
			select {
			// 优雅退出
			case <-b.GetStopChan():
				close(deliveries)
				return
         // 消费令牌
			case <-pool:
				// 获取消息 BLPOP
				task, _ := b.nextTask(getQueue(b.GetConfig(), taskProcessor))
				if len(task) > 0 {
               // 异步发送到任务队列
					deliveries  <- task
				}

            // 归还令牌
				pool <- struct{}{}
			}
		}
	}()
   //...
}
```

##    0x05  一些细节
版本锁的 bug：https://github.com/RichardKnop/machinery/commit/d8e5bcf7469429aaeb3346499e8a278924c44e53

##    0x06  总结
基于 [前文](https://pandaychen.github.io/2022/01/01/A-KAFKA-USAGE-SUMUP-2/) 中分析 Kafka 的生产消费可靠模型，Machinery 框架有 4 个组件，Server 负责任务的生产，Broker 负责任务的分发，Worker 负责任务的执行，Backend 负责存储任务的状态。Server 确保任务发布到 Broker 中了，Broker 确保任务被成功消费了，Worker 确保任务会被成功执行。**Machinery 不确保 Worker 能顺利执行完任务，只能确保 Worker 能取到任务**


## 0x07  参考
-  [machinery 中文文档](https://zhuanlan.zhihu.com/p/270640260)
-  [Golang 任务队列 machinery 使用与源码剖析（一）](https://cloud.tencent.com/developer/article/1169675)
- [Golang 任务队列 machinery 使用与源码剖析（一）](https://cloud.tencent.com/developer/article/1169675)
- [machinery](https://github.com/RichardKnop/machinery)
- [feat: Kafka broker](https://github.com/RichardKnop/machinery/pull/449/files#)
- [有赞延迟队列设计](https://tech.youzan.com/queuing_delay/)
- [分布式任务队列 machinery 的使用](https://www.lijiaocn.com/%E7%BC%96%E7%A8%8B/2017/11/06/go-async-tasks.html)