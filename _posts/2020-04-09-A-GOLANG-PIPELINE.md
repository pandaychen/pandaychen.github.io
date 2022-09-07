---
layout:     post
title:      Golang 并发模型：Pipelines and cancellation
subtitle:   Channel 的经典应用
date:       2020-04-09
author:     pandaychen
header-img: img/golang-horse-fly.png
catalog: true
category:   false
tags:
    - Golang
    - 并发
---

##  0x00    前言
[Go Concurrency Patterns: Pipelines and cancellation](https://blog.golang.org/pipelines) 一文介绍了 Golang 的最经典的并发及控制模型：Pipeline（流水线模型），基于 Channel 的生产者消费者模型。本文即整理及总结。

文中给出了几个典型 goroutine 的并发及控制场景：
1.  **Fan-out-fan-in**：扇入扇出

##  0x01    什么是 Pipeline
Pipeline（流水线）是由多个阶段组成的，相邻的两个阶段由 channel 进行连接；每个阶段是由一组在同一个函数中启动的 goroutine 组成。在每个阶段，这些 goroutine 会执行下面三个操作：
1.  通过 Inbound Channels 从上游接收数据
2.  对接收到的数据执行一些操作，通常会生成新的数据
3.  将新生成的数据通过 Outbound Channels 发送给下游
除了第一个和最后一个阶段，每个阶段都可以有任意个 Inbound 和 Outbound Channel，有点 Map-Reduce 的味道。本质上还是生产者 - 消费者的模型。


##  0x02    case1：计算平方数

| 阶段 | 功能 | 方法 |
|------|------------|----------|
| 阶段 1   |  传入输入     | gen|
| 阶段 2   |   计算平方      |     sq  |
| 阶段 3 ||main 打印结果 |

代码如下：
阶段 1：创建输入参数为可变长 int 整数的 gen 函数，它通过 goroutine 发送所有输入参数，并在发送完成后关闭相应 channel<br>
```golang
func gen(nums ...int) <-chan int {
	out := make(chan int)
	go func() {
		for _, n := range nums {
			out <- n
		}
		close(out)
	}()

	return out
}
```
阶段 2：sq 函数，负责从 Inbound channel 中接收数据并作平方处理再发送到 Outbound channel 中。在输入 channel 关闭并把所有数据都成功发送至 Outbound Channel，关闭输出 channel<br>
```golang
func sq(in <-chan int) <-chan int {
	out := make(chan int)
	go func() {
		for n := range in {
			out <- n * n
		}
		close(out)
	}()

	return out
}
```

阶段 3：主函数 main 中创建 pipeline 并执行了最后阶段的任务，即从 Outbound Channel 接收数据并打印出来 <br>
```golang
func main() {
	// Set up the pipeline
	c := gen(2, 3)
	out := sql(c)

	// Consume the output
	fmt.Println(<-out)
    fmt.Println(<-out)
    /*
    // Set up the pipeline and consume the output.
	for n := range sq(sq(gen(2, 3))) {
		fmt.Println(n)
    }
    */
}
```

##	0x03	case2：分块计算MD5

##	0x03	case3：

##  0x04  参考
-   [Go Concurrency Patterns: Pipelines and cancellation](https://blog.golang.org/pipelines)
-   [Pipeline Patterns in Go：Pipelines with Error-handling and Cancellation](https://medium.com/statuscode/pipeline-patterns-in-go-a37bb3a7e61d)
-	[pipelines](https://go.dev/blog/pipelines)