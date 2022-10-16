---
layout: post
title: Golang 的分布式任务队列：Asynq 分析
subtitle: 
date: 2021-08-18
author: pandaychen
catalog: true
tags:
    - Asynq
    - 队列
    - 异步队列
---


##  0x00    前言
[asynq](https://github.com/hibiken/asynq)：Golang distributed task queue library，许多设计思想都来自[sidekiq](https://github.com/mperham/sidekiq)。[官方文档](https://github.com/hibiken/asynq/wiki)给出的特点如下（省略了若干）：

-   任务已写入Redis后会持久化（支持Redis Cluster/Sentinels）
-   任务执行失败自动 retry
-   支持任务优先级权重（加权优先级队列）
-   支持定时发送任务
-   可使用 unique-option 來避免任务重复执行
-   UI及客户端、metrics支持（CLI检查和远程控制队列和任务）
-   支持任务设置执行时间or最长可运行的时间

库的使用示例[在此](https://github.com/hibiken/asynq/wiki/Getting-Started)

##  0x01    代码分析
asynq整体流程上和先前分析的中间件大同小异，并且该中间件严重依赖Redis，所有的原子化逻辑都是通过lua脚本实现的（如任务的出入队列等等），按照任务的状态划分了不同的redis数据结构及[逻辑](https://github.com/hibiken/asynq/blob/master/internal/rdb/rdb.go)：

-   active：asynq:{<qname>}:active LIST类型
-   pending：asynq:{<qname>}:pending    LIST类型
-   lease：asynq:{<qname>}:lease SortedSet类型
-   completed：asynq:{<qname>}: SortedSet类型
-   paused：asynq:{<qname>}:paused：HASHTABLE类型

此外，还有各种不同类型的结构，用来记录任务执行的状态、计数器等等

####    任务操作的Redis封装
1、[`dequeueCmd`](https://github.com/hibiken/asynq/blob/master/internal/rdb/rdb.go#L204)：用于把任务从`pending`List取出，放到`active`List中
```GOLANG
// Input:
// KEYS[1] -> asynq:{<qname>}:pending
// KEYS[2] -> asynq:{<qname>}:paused
// KEYS[3] -> asynq:{<qname>}:active
// KEYS[4] -> asynq:{<qname>}:lease
// --
// ARGV[1] -> initial lease expiration Unix time
// ARGV[2] -> task key prefix
//
// Output:
// Returns nil if no processable task is found in the given queue.
// Returns an encoded TaskMessage.
//
// Note: dequeueCmd checks whether a queue is paused first, before
// calling RPOPLPUSH to pop a task from the queue.
var dequeueCmd = redis.NewScript(`
if redis.call("EXISTS", KEYS[2]) == 0 then
	local id = redis.call("RPOPLPUSH", KEYS[1], KEYS[3])
	if id then
		local key = ARGV[2] .. id
		redis.call("HSET", key, "state", "active")
		redis.call("HDEL", key, "pending_since")
		redis.call("ZADD", KEYS[4], ARGV[1], id)
		return redis.call("HGET", key, "msg")
	end
end
return nil`)
```

2、`markAsCompleteCmd`方法：任务成功处理完成，通过`HSET`更新状态，同时删除`active`LIST的数据
```GOLANG
// KEYS[1] -> asynq:{<qname>}:active
// KEYS[2] -> asynq:{<qname>}:lease
// KEYS[3] -> asynq:{<qname>}:completed
// KEYS[4] -> asynq:{<qname>}:t:<task_id>
// KEYS[5] -> asynq:{<qname>}:processed:<yyyy-mm-dd>
// KEYS[6] -> asynq:{<qname>}:processed
//
// ARGV[1] -> task ID
// ARGV[2] -> stats expiration timestamp
// ARGV[3] -> task exipration time in unix time
// ARGV[4] -> task message data
// ARGV[5] -> max int64 value
var markAsCompleteCmd = redis.NewScript(`
if redis.call("LREM", KEYS[1], 0, ARGV[1]) == 0 then
  return redis.error_reply("NOT FOUND")
end
if redis.call("ZREM", KEYS[2], ARGV[1]) == 0 then
  return redis.error_reply("NOT FOUND")
end
if redis.call("ZADD", KEYS[3], ARGV[3], ARGV[1]) ~= 1 then
  redis.redis.error_reply("INTERNAL")
end
redis.call("HSET", KEYS[4], "msg", ARGV[4], "state", "completed")
local n = redis.call("INCR", KEYS[5])
if tonumber(n) == 1 then
	redis.call("EXPIREAT", KEYS[5], ARGV[2])
end
local total = redis.call("GET", KEYS[6])
if tonumber(total) == tonumber(ARGV[5]) then
	redis.call("SET", KEYS[6], 1)
else
	redis.call("INCR", KEYS[6])
end
return redis.status_reply("OK")
`)
```

####    核心结构
1、`processor`结构<br>
```GOLANG
type processor struct {
	logger *log.Logger
	broker base.Broker
	clock  timeutil.Clock

	handler   Handler
	baseCtxFn func() context.Context

	queueConfig map[string]int

	// orderedQueues is set only in strict-priority mode.
	orderedQueues []string

	retryDelayFunc RetryDelayFunc
	isFailureFunc  func(error) bool

	errHandler ErrorHandler

	shutdownTimeout time.Duration

	// channel via which to send sync requests to syncer.
	syncRequestCh chan<- *syncRequest

	// rate limiter to prevent spamming logs with a bunch of errors.
	errLogLimiter *rate.Limiter

	// sema is a counting semaphore to ensure the number of active workers
	// does not exceed the limit.
	sema chan struct{}

	// channel to communicate back to the long running "processor" goroutine.
	// once is used to send value to the channel only once.
	done chan struct{}
	once sync.Once

	// quit channel is closed when the shutdown of the "processor" goroutine starts.
	quit chan struct{}

	// abort channel communicates to the in-flight worker goroutines to stop.
	abort chan struct{}

	// cancelations is a set of cancel functions for all active tasks.
	cancelations *base.Cancelations

	starting chan<- *workerInfo
	finished chan<- *base.TaskMessage
}

```


####    核心流程
1、`processor.exec`<br>
asynq没有使用 `brpop/rpop` 指令（前者阻塞，后者轮询），而是通过 channel 实现信号量去控制对 Redis 的轮询。参见下面的`processor.exec`方法，该方法独立运行，会先尝试获取一个允许继续执行的信号，如果有则调用 broker的`Dequeue` 方法查询待执行的任务信息，否则就停下来等待信号。如果队列是空的，那么会`time.Sleep(time.Second)`，以免不断的查询 Redis 。此外，执行信号的个数也是可配置（默认机器核数），这里就是普通的限制取任务并发

```GOLANG
// exec pulls a task out of the queue and starts a worker goroutine to
// process the task.
func (p *processor) exec() {
	select {
	case <-p.quit:
		return
	case p.sema <- struct{}{}: // acquire token
		qnames := p.queues()

        //从broker中取任务数据
		msg, leaseExpirationTime, err := p.broker.Dequeue(qnames...)
		switch {
		case errors.Is(err, errors.ErrNoProcessableTask):
			p.logger.Debug("All queues are empty")
			// Queues are empty, this is a normal behavior.
			// Sleep to avoid slamming redis and let scheduler move tasks into queues.
			// Note: We are not using blocking pop operation and polling queues instead.
			// This adds significant load to redis.
			time.Sleep(time.Second)
			<-p.sema // release token
			return
		case err != nil:
			if p.errLogLimiter.Allow() {
				p.logger.Errorf("Dequeue error: %v", err)
			}
			<-p.sema // release token
			return
		}

		lease := base.NewLease(leaseExpirationTime)
		deadline := p.computeDeadline(msg)
		p.starting <- &workerInfo{msg, time.Now(), deadline, lease}
		go func() {
			defer func() {
				p.finished <- msg
				<-p.sema // release token
			}()

			ctx, cancel := asynqcontext.New(p.baseCtxFn(), msg, deadline)
			p.cancelations.Add(msg.ID, cancel)
			defer func() {
				cancel()
				p.cancelations.Delete(msg.ID)
			}()

			// check context before starting a worker goroutine.
			select {
			case <-ctx.Done():
				// already canceled (e.g. deadline exceeded).
				p.handleFailedMessage(ctx, lease, msg, ctx.Err())
				return
			default:
			}

			resCh := make(chan error, 1)
			go func() {
				task := newTask(
					msg.Type,
					msg.Payload,
					&ResultWriter{
						id:     msg.ID,
						qname:  msg.Queue,
						broker: p.broker,
						ctx:    ctx,
					},
				)
				resCh <- p.perform(ctx, task)
			}()

			select {
			case <-p.abort:
				// time is up, push the message back to queue and quit this worker goroutine.
				p.logger.Warnf("Quitting worker. task id=%s", msg.ID)
				p.requeue(lease, msg)
				return
			case <-lease.Done():
				cancel()
				p.handleFailedMessage(ctx, lease, msg, ErrLeaseExpired)
				return
			case <-ctx.Done():
				p.handleFailedMessage(ctx, lease, msg, ctx.Err())
				return
			case resErr := <-resCh:
				if resErr != nil {
					p.handleFailedMessage(ctx, lease, msg, resErr)
					return
				}
				p.handleSucceededMessage(lease, msg)
			}
		}()
	}
}
```

2、任务执行<br>
任务的执行也是异步的
```GOLANG
func (p *processor) exec() {
    //...
    go func() {
                    task := newTask(
                        msg.Type,
                        msg.Payload,
                        &ResultWriter{
                            id:     msg.ID,
                            qname:  msg.Queue,
                            broker: p.broker,
                            ctx:    ctx,
                        },
                    )
                    resCh <- p.perform(ctx, task)
                }()

    //...
}
// perform calls the handler with the given task.
// If the call returns without panic, it simply returns the value,
// otherwise, it recovers from panic and returns an error.
func (p *processor) perform(ctx context.Context, task *Task) (err error) {
	defer func() {
		if x := recover(); x != nil {
			p.logger.Errorf("recovering from panic. See the stack trace below for details:\n%s", string(debug.Stack()))
			_, file, line, ok := runtime.Caller(1) // skip the first frame (panic itself)
			if ok && strings.Contains(file, "runtime/") {
				// The panic came from the runtime, most likely due to incorrect
				// map/slice usage. The parent frame should have the real trigger.
				_, file, line, ok = runtime.Caller(2)
			}

			// Include the file and line number info in the error, if runtime.Caller returned ok.
			if ok {
				err = fmt.Errorf("panic [%s:%d]: %v", file, line, x)
			} else {
				err = fmt.Errorf("panic: %v", x)
			}
		}
	}()
	return p.handler.ProcessTask(ctx, task)
}

func (fn HandlerFunc) ProcessTask(ctx context.Context, task *Task) error {
	return fn(ctx, task)
}
```

3、等待任务执行结果<br>
```GOLANG
func (p *processor) exec() {
    //...
    select {
			case <-p.abort:
				// time is up, push the message back to queue and quit this worker goroutine.
				p.logger.Warnf("Quitting worker. task id=%s", msg.ID)
				p.requeue(lease, msg)
				return
			case <-lease.Done():
				cancel()
				p.handleFailedMessage(ctx, lease, msg, ErrLeaseExpired)
				return
			case <-ctx.Done():
				p.handleFailedMessage(ctx, lease, msg, ctx.Err())
				return
			case resErr := <-resCh:
				if resErr != nil {
					p.handleFailedMessage(ctx, lease, msg, resErr)
					return
				}
				p.handleSucceededMessage(lease, msg)
			}
    //...
}
```
##  0x02    总结


##  0x03 参考
-   [如何实现一个任务队列](https://chordl.me/2021/how-to-build-a-task-queue-7bed4aef)