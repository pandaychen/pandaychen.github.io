---
layout:     post
title:      一个轻量级的 golang 协程池的实现
subtitle:   使用 Golang-channel 实现工作池：批量并发处理
date:       2020-07-01
author:     pandaychen
header-img: img/golang-horse-fly.png
catalog:    true
tags:
    - Golang
---


##  0x00    前言
网络上有非常多优秀的协程池实现及原理介绍，推荐阅读下面的链接：
-   fasthttp 的 [协程池实现](https://github.com/valyala/fasthttp/blob/master/workerpool.go)
-   [Goroutine 并发调度模型深度解析之手撸一个高性能 goroutine 池](https://taohuawu.club/high-performance-implementation-of-goroutine-pool)

本文介绍如何实现 goroutine`+`Channel 机制实现一套通用的协程池，用来控制并发执行的安全性，同时提升效率。

##  0x01    Goroutine 高并发的问题
Golang 原生支持 goroutine 并发，可以同时启动多个 goroutine。如何管理这些 goroutine 的启动停止就是一个问题，事实上，goroutine 并非越多越好。从目前工作中遇到的问题来看，有以下几点需要特别注意的：
-   多 goroutine 场景下导致系统资源的耗尽
-   多 goroutine 场景（高并发）下的 gc STW 延迟增大的问题
-   多 goroutine 下的上下文切换的延迟

总结下，goroutine 并非越多越好，协程的数量和执行效率之间往往存在一个平衡点，如果控制不好，极有可能因资源争抢而出现阻塞，或因对外部服务的高频访问而将对端服务拖死。应对此类问题的一种解决思路就是：goroutine Pool。

##  0x02    设计目标 && 思路
1.  协程的数量可指定
2.  返回结果状态
3.  工作并发控制，当前运行工作数 `<=` 最大协程数

在 Golang 中，Channel 是语言级支持的一种数据类型，实现了协程间基于消息传递的通信方式，是线程安全的。这里先将原始的任务数据以某种统一的数据结构 (RawTask) 统一推入到 Channel 管道中，即形成了一个原任务 Channel。
启动指定数量的工作协程，对 Channel 中的任务数据进行抢占式消费，即在所有的工作协程中，谁抢到任务谁处理，处理完成后，将处理结果以某种统一的数据结构 (RetTask) 写入另一个结果 Channel 中，然后再继续获取原任务进行执行，如果工作协程发现原任务队列已为空，则协程退出。
这种方式不会给任何工作协程分配指定数量的任务，这样效率高的可以多处理，效率低的允许少处理，从总体上达到处理时间最小化。
等待所有任务处理完成后，将处理结果数据统一返回上到层调用逻辑。

####    核心方法实现

`TaskInput` 为单个待处理任务的结构：
```golang
type TaskInput struct {
	Id   string     //ID 用来标识任务
	Guid string
	InputData interface{}       // 与任务关联的数据
}
```

`TaskResult` 为用于返回单个任务结果的结构：

```GOLANG
type TaskResult struct {
	Id         string // 回调 ID
	Guid       string
	OutputData interface{}  // 结果
	Err        error
	Boolret    bool
}
```

`DoFixedSizeWorks` 为核心方法，实现了并发协程池的逻辑：

```golang
var ERROR_PARAM_ERROR = errors.New("Pool len illegal")

func DoFixedSizeWorks(concurrency int, TaskInputList []TaskInput, fun func(task TaskInput) (id, guid string, outputData interface{}, err error, boolret bool)) ([]TaskResult, error) {
	ResultList := make([]TaskResult, 0)
	taskLen := len(TaskInputList)
	if taskLen == 0 {
		return nil, ERROR_PARAM_ERROR
    }

	chInputDataList := make(chan TaskInput, taskLen)
	for _, task := range TaskInputList {
		chInputDataList <- task
	}

	chResultList := make(chan TaskResult, taskLen)
	var lock sync.RWMutex

    // 启动 concurrency 个协程
	for i := 0; i < concurrency; i++ {
		go func() {
			for {
				lock.Lock()
				if len(chInputDataList) == 0 {
					lock.Unlock()
					break
				}
				taskData := <-chInputDataList
                lock.Unlock()

                // 调用用户传入的方法（回调）
				id, guid, outputData, err, boolret := fun(taskData)
				chResultList <- TaskResult{
					Id:         id,
					Guid:       guid,
					Err:        err,
					OutputData: outputData, Boolret: boolret}
			}
		}()
	}

	// 阻塞接收协程的执行结果（从 <-chResultList 中获取结果）
	for i := 0; i < taskLen; i++ {
		ResultList = append(ResultList, <-chResultList)
	}
	return ResultList, nil
}
```

##  0x03    调用实例
调用协程池的方法也非常简单，首先，自定义我们的工作处理回调方法 `WorkCallback`：
```golang
func WorkCallback(rawTask TaskInput) (id, guid string, outputData interface{}, err error, ret bool) {
	//DO Batch Jobs
	fmt.Println(rawTask)
	outputData = map[string]string{"id": rawTask.Id, "guid": rawTask.Guid}
	fmt.Printf("Doing Work...id=%s,guid=%s\n", rawTask.Id, rawTask.Guid)
	time.Sleep(10 * time.Second)
	ret = true
	return rawTask.Id, rawTask.Guid, outputData, nil, ret
}
```

用协程池来模拟工作的代码如下：
```golang
func init() {
	// 以时间作为初始化种子
	rand.Seed(time.Now().UnixNano())
}

func main() {
	worknum := flag.Int("worknum", 0, "worker num")
	flag.Parse()

	if *worknum <= 0 {
		fmt.Println("Usage Error!")
		return
	}

    jobsize := 200
    // 模拟向任务队列添加 jobsize 个任务
	tasklist := make([]TaskInput, 0)
	for i := 0; i < jobsize; i++ {
		u, err := uuid.NewV4()
		if err != nil {
			continue
		}
		input := &TaskInput{
			Id:   strconv.Itoa(i),
			Guid: string(u.String())}
		tasklist = append(tasklist, *input)
	}

	output, _ := DoFixedSizeWorks(*worknum, tasklist, WorkCallback)

	for _, v := range output {
		fmt.Println("result=", v.Id, v.OutputData)
	}
}
```

完整的 [实现代码在此](https://github.com/pandaychen/go-worker-pool/blob/master/batch_wpool.go)

##  0x04    参考
-   [容器中某 Go 服务 GC 停顿经常超过 100ms 排查](http://www.dockone.io/article/9387)
-   [Goroutine 并发调度模型深度解析之手撸一个高性能 goroutine 池](https://taohuawu.club/high-performance-implementation-of-goroutine-pool)

