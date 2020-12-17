---
layout:     post
title:      基于 Golang 实现的定时器分析与实现（未完成）
subtitle:   Timer and Ticker: events in the future
date:       2020-06-01
author:     pandaychen
catalog:    true
header-img: img/panda-md-pic7.jpg
tags:
    - Golang
    - 定时器
    - Timer
    - Ticker
---


##  0x00    前言

##  0x01    Golang 原生库的使用

##  0x02    原生库的实现

##  0x03    最小堆的实现
最小堆（minheap）定时器的经典用法是和 [`epoll_wait` 系统调用](https://man7.org/linux/man-pages/man2/epoll_wait.2.html) 的结合，即与参数 `timeout` 结合：
![epoll_wait](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/linux/epoll_wait.png)
`timeout` 表示超时时间（一般的实例都是设置一个固定的值），但是可以通过与最小堆结合，在事件驱动循环中动态计算 `timeout` 的值。具体做法是：通过小根堆保存每个超时事件，将 `timeout` 设置为 top 结点（即堆顶）的时间，就完美的将网络事件 IO 与超时事件结合在一起。<br>
具体的示例代码可以参见：[min_heap.c](https://github.com/pandaychen/tcpframe/blob/main/src/min_heap.c)

![epoll_min_heap](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/linux/epoll_min_heap.jpg)

##  0x04    Timewheel 实现

##  0x05    其他

##  0x06    参考
-   [Go-Zero 如何应对海量定时 / 延迟任务](https://my.oschina.net/u/4628563/blog/4667586)
-   [论 golang Timer Reset 方法使用的正确姿势](https://tonybai.com/2016/12/21/how-to-use-timer-reset-in-golang-correctly/)

