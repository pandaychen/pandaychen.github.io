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
---

##  0x00    前言
一个现实的问题，在调用 `io.Copy()` 进行数据传输时，能够有效的控制传输的速率吗？有几个前提：
- 相比限流直接了当的给一个错误的返回值，限速要做的不仅不能返回错误，而且要在一定的时间段内进行响应，否则可能造成超时



##  参考
-   [从 io.Reader 中读数据](https://colobu.com/2019/02/18/read-data-from-net-Conn/)
