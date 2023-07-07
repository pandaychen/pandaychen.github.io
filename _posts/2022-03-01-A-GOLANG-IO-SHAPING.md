---
layout:     post
title:      数据结构与算法回顾（六）：Golang IO shaping
subtitle:   基于限流器的流量整形算法实现
date:       2022-03-01
author:     pandaychen
header-img:
catalog: true
tags:
    - Golang
    - 数据结构
---

##  0x00    前言
关于流量整形，接触过两种不同的概念：
1.	DDoS 的限速（流），按照流量或者 QPS 进行限制，超出的部分丢弃
2.	SCP 工具传输的限速（FTP 数据源端限速），如 `scp -l 1000 file user@remote:/path/to/dest/S`，仅限制速率恒定，不丢弃

本文讨论一个现实的问题，在调用 `io.Copy()` 进行数据传输时，能够有效的控制传输的速率吗？有几个前提：
-	相比于一般限流触发时给一个错误的返回值，这里讨论的流量整形不仅不能返回错误，而且要在一定的时间段内进行响应，否则可能造成超时
-	限流的速率要保持恒定

##	0x01	背景
以网络环境而言，流量整形是对输出报文的速率进行控制，使报文以均匀的速率发送出去。通常是为了使报文速率与下游设备相匹配。当从高速链路向低速链路传输数据，或发生突发流量时，带宽会在低速链路出口处出现瓶颈，导致数据丢失严重。于是就有了流量整形，通过在上游设备（高速）的接口出方向配置流量整形，将上游不规整的流量进行削峰填谷，输出一条比较平整的流量，从而解决下游设备的瞬时拥塞问题。

![shaping-1](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/io/io-shaping-1.png)

流量整形的效果如下：
![shaping-2](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/io/io-shaping-2.png)

##	0x02	流量整形的原理
流量整形功能通常使用缓冲区和令牌桶来完成，当报文的发送速度过快时，首先在缓冲区进行缓存，在令牌桶的控制下再均匀地发送这些被缓冲的报文（考虑令牌桶是匀速的添加）。采用接口流量整形的思路实现：限制接口发送的所有报文（包括紧急报文）的总速率，是对整个出接口进行流量整形，不区分优先级。
1.	队列调度之后在出队的时候，对所有队列的数据包总和进行令牌桶评估
2.	在令牌桶评估后，如果数据包总速率符合要求，则转发；如果数据包总速率超标（即令牌桶中的令牌不足），则接口暂停调度，等令牌足够时再继续调度

![shaping-3](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/io/io-shaping-3.png)


##	0x03	特殊的令牌桶
由于流量的速度一般是 `Kb`、`Mb` 等，所以令牌桶的速率也要考虑限流的速度。不仅需要限制读取数据量的大小，而且要均衡到一定的时间窗口上：

-	设定合适的限流单位区间（如 `1ms`、`10ms` 等），防止时间窗口过大导致的限速不合理，有点像滑动窗口
-	单位区间限速的大小，过大或者过小都是不合理的
-	令牌桶初始化时应该被清空（思考下为何）
-	控制采用 `time.Sleep(duration)`，duration 精确到单位区间的倍数，`Sleep` 的时间只要能够产生指定的 token 数量就行

举例来说，假设单位区间为 `10ms`，每个单位区间允许的字节数是 `N`，那么 `1s` 对应 `100` 个 `10ms`，那每 `1s` 对应的流速就是 `N*100`，不过这 `N*100` 是均匀的散落在 `100` 个单位区间里面的

或者有另外一种思路，直接使用时间区间的大小作为 token 的数量，每个 token 对应一个流量（如 `256Kb`），假设 `1s` 可允许 `10` 个 token，那么 `100ms` 仅能获取 `1` 个 token，`50ms` 只能获取 `0.5` 个 token

##  0x04    代码分析
代码实现在[此](https://github.com/pandaychen/goes-wrapper/blob/master/pyio/limit_io.go)，核心是采用令牌桶算法实现。


####    测试
测试传输文件大小为`6.3M`，限制传输速度为`512KB/s`，测试结果符合设计：
```text
[root@VM_120_245_centos ~/io]# time ./limit_io

real    0m12.861s
user    0m0.044s
sys     0m0.004s

[root@VM_120_245_centos ~/io]# md5sum test_file destFile 
a002ec21223b0c402a8310b5ec778190  test_file
a002ec21223b0c402a8310b5ec778190  destFile
```

##  0x05    参考
-   [从 io.Reader 中读数据](https://colobu.com/2019/02/18/read-data-from-net-Conn/)
-	[流量整形](https://support.huawei.com/enterprise/zh/doc/EDOC1100143291/4395b710)