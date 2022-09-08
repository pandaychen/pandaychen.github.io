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
    - Pipeline
---

##  0x00    前言
[Go Concurrency Patterns: Pipelines and cancellation](https://blog.golang.org/pipelines) 一文介绍了 Golang 的最经典的并发及控制模型：Pipeline（流水线模型），基于 Channel 的生产者消费者模型。本文即整理及总结。

文中给出了几个典型 goroutine 的并发及控制场景：
1.  Squaring numbers：计算平方数
2.	Fan-out-fan-in：扇入扇出
3.	Stopping short
4.	Explicit cancellation
5.	Digesting a tree
6.	Parallel digestion
7.	Bounded parallelism


##  0x01    什么是 Pipeline
Pipeline（流水线）是由多个阶段组成的，相邻的两个阶段由 channel 进行连接；每个阶段是由一组在同一个函数中启动的 goroutine 组成。在每个阶段，这些 goroutine 会执行下面三个操作：
1.  通过 Inbound Channels 从上游接收数据
2.  对接收到的数据执行一些操作，通常会生成新的数据
3.  将新生成的数据通过 Outbound Channels 发送给下游
除了第一个和最后一个阶段，每个阶段都可以有任意个 Inbound 和 Outbound Channel，有点 Map-Reduce 的味道。本质上还是生产者 - 消费者的模型。


##  0x02    case1：计算平方数

| 阶段 | 功能 | 方法 |
|------|------------|----------|
| 阶段 `1`   |  传入输入     | `gen`|
| 阶段 `2`   |   计算平方      |     `sq`  |
| 阶段 `3` ||`main` 打印结果 |

代码如下：
阶段 1：创建输入参数为可变长 `int` 整数的 `gen` 函数，它通过 goroutine 发送所有输入参数，并在发送完成后关闭相应 channel<br>
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
阶段 2：`sq` 函数，负责从 Inbound channel 中接收数据并作平方处理再发送到 Outbound channel 中。在输入 channel 关闭并把所有数据都成功发送至 Outbound Channel，关闭输出 channel<br>
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

阶段 3：主函数 `main` 中创建 pipeline 并执行了最后阶段的任务，即从 Outbound Channel 接收数据并打印出来 <br>
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

由于`sq`的入站和出站channel类型一样，所以可以多次组合使用它：
```golang
func main() {
    for n := range sq(sq(gen(2, 3))) {
        fmt.Println(n)
    }
}
```

##	0x03	Fan-out-fan-in：扇出，扇入
-	扇出：多个方法可以从同一个channel读取数据，直到该channel被关闭。扇出提供了一种分发任务，从而并行化使用CPU和I/O的方式（不过注意一点：这里多个goroutine同时访问一个channel时，由runtime来调度哪个goroutine来获取数据，不需要开发者实现）
-	扇入：将多个输入channel复用到单个channel上，在所有输入channel关闭时，关闭这个channel。通过这种方式，一个函数可以从多个输入读取数据，并执行相应的操作直到所有的数据源都被关闭

继续上面的例子，运行两个`sq`，每个`sq`都从同一个输入channel读取数据。引入`merge`方法扇入结果，如下（通过`merge`接收多个channel的数据，写入一个结果channel）：

```golang
func merge(cs ...<-chan int) <-chan int {
    var wg sync.WaitGroup
    out := make(chan int)
    
    // 为每个入channel启动一个名为output的goroutine
    // output从每个channel中读取值并传入out，直到入channel被关闭，然后调用wg.Done
    output := func(c <-chan int) {
        for n := range c {
            out <- n
        }
        //for range可感知channel的退出！
        wg.Done()
    }

    //对每个chan int都单独启动goroutine，将chan中的数据取出来放在out里面
    wg.Add(len(cs))
    for _, c := range cs {
        go output()
    }
    
    // 当所有的output 的goroutine都执行完成时，启动一个新的goroutine来关闭out
    // 必须在调用wg.Add后执行这个操作。

    //回收操作
    go func() {
        wg.Wait()
        //特别处理close(out)的时机
        close(out)
    }()
    return out
}

func main() {
    in := gen(2, 3)
    
    // Distribute the sq work across two goroutines that both read from in.
    c1 := sq(in)
    c2 := sq(in)
    
    // Consume the merged output from c1 and c2.
    for n := range merge(c1, c2) {
        fmt.Println(n) // 4 then 9, or 9 then 4
    }
}
```

上面这个例子给出了很典型的经验汇总：
1.  `sync.WaitGroup`管理goroutine正常退出
2.  向一个已经关闭的channel传入值会导致程序崩溃，所以在关闭channel前一定要保证所有的传值操作都已完成
3.  利用channel做结果传输的通道


##	0x04	case2：分块计算MD5


##  0x0  参考
-   [Go Concurrency Patterns: Pipelines and cancellation](https://blog.golang.org/pipelines)
-   [Pipeline Patterns in Go：Pipelines with Error-handling and Cancellation](https://medium.com/statuscode/pipeline-patterns-in-go-a37bb3a7e61d)
-	[pipelines](https://go.dev/blog/pipelines)