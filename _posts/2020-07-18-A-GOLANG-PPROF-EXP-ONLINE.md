---
layout:     post
title:      项目中golang相关的优化 Case 处理及解决
subtitle:   如何使用 Golang 的 Pprof 工具进行内存分析以及 sync.Pool 优化
date:       2020-07-18
author:     pandaychen
catalog:    true
tags:
    - Pprof
---

##  0x00    前言
pprof是Go的性能分析工具，在程序运行过程中，可以记录程序的运行信息，如CPU使用情况、内存使用情况、goroutine运行情况等。分析性能需要结合具体的场景来看

####    性能指标
-   吞吐量：每秒钟可以处理的请求数
-   响应时间：从客户端发出请求，到收到回包的总耗时
-   内存使用率
-   CPU使用率
-   IO使用率

##  0x01    工具介绍
golang可通过`benchmark`加`pprof`来定位具体的性能瓶颈。

####    benchmark
常用指令：`go test -v some_code_test.go -run=none -bench=. -benchtime=3s -cpuprofile cpu.prof -memprofile mem.prof`

-   `-run` ：单次测试，一般用于代码逻辑验证
-   `-bench=.`：执行所有Benchmark，也可以通过用例函数名来指定部分测试用例
-	`-benchtime`：指定测试执行时长
-	`-cpuprofile`： 输出cpu的pprof信息文件
-	`-memprofile`：输出heap的pprof信息文件
-	`-blockprofile`：阻塞分析，记录 goroutine 阻塞等待同步（包括定时器通道）的位置
-	`-mutexprofile`：互斥锁分析，报告互斥锁的竞争情况

benchmark测试用例常用函数
-	`b.ReportAllocs()`：输出单次循环使用的内存数量和对象`allocs`信息
-	`b.RunParallel()`：使用协程并发测试
-	`b.SetBytes(n int64)`：设置单次循环使用的内存数量

####    pprof
1、生成方式通常有如下几种<br>
-   `runtime/pprof`: 手动调用如`runtime.StartCPUProfile`或者`runtime.StopCPUProfile`等 API来生成和写入采样文件，灵活性高。主要用于本地测试
-   `net/http/pprof`: 通过 http 服务（主代码内嵌）获取Profile采样文件，适用于对应用程序的整体监控。通过 `runtime/pprof` 实现。主要用于服务器端测试
-   `go test`: 通过 `go test -bench . -cpuprofile cpuprofile.out`生成采样文件，主要用于本地基准测试。可用于重点测试某些函数


2、查看方式<br>
常用指令1：`go tool pprof [options] [binary] -o xxxx`<br>

输出格式: 一般常用的是：
-   `–text`：纯文本
-	`–web`：生成svg并用浏览器打开（如果svg的默认打开方式是浏览器)
-	`–svg`：只生成svg
-	`–list funcname`：筛选出正则匹配funcname的函数的信息
-   `-http=":port"`：直接本地浏览器打开profile查看（包括top，graph，火焰图等）

常用指令2：`go tool pprof -base profile1 profile2`，该指令意义为对比查看2个profile，一般用于代码修改前后对比，定位差异点

命令行方式：通过命令行方式查看profile时，通常使用指令：
1)、`topN [-cum]` 查看前`N`个数据<br>

```text
flat  flat%   sum%        cum   cum%
5.95s 27.56% 27.56%      5.95s 27.56%  runtime.usleep
4.97s 23.02% 50.58%      5.08s 23.53%  sync.(*RWMutex).RLock
4.46s 20.66% 71.24%      4.46s 20.66%  sync.(*RWMutex).RUnlock
2.69s 12.46% 83.70%      2.69s 12.46%  runtime.pthread_cond_wait
1.50s  6.95% 90.64%      1.50s  6.95%  runtime.pthread_cond_signal
```

字段的意义为：

| 字段 | 意义 |
| :-----:| :----: |
| flat | 采样时，该函数正在运行的次数*采样频率（10ms），即得到估算的函数运行采样时间。这里不包括函数等待子函数返回 | 
| flat% | flat / 总采样时间值 |
| sum%| 前面所有行的 flat% 的累加值，如第三行 sum% = 71.24% = 27.56% + 50.58% | 
| cum| 采样时，该函数出现在调用堆栈的采样时间，包括函数等待子函数返回。因此 flat <= cum | 
| cum%| cum / 总采样时间值|

2)、`list funcname`:查看某个函数的详细信息，可以明确具体的资源（cpu，内存等）是由哪一行触发的<br>


##  0x02    优化经验

##  0x02    具体Case1


##  0x  参考
-   [go pprof 性能分析](https://juejin.cn/post/6844903588565630983)