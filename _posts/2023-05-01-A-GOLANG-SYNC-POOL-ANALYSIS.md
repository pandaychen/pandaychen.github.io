---
layout:     post
title:      Sync.Pool 应用与分析（续）
subtitle:   Golang sync.Pool 源码分析
date:       2023-05-01
author:     pandaychen
catalog:    true
tags:
    - Golang
    - Pool
---


##  0x00    前言

前文 [Golang 中的 sync.Pool 使用](https://pandaychen.github.io/2020/03/11/GOLANG-SYNC-POOL-USAGE/) 介绍了 `sync.Pool` 的使用及若干细节。

`sync.Pool` 是并发安全且可伸缩的，常用于存放可复用的对象的一个容器，以减少反复创建对象带来的开销，这里需要注意，Pool 是用于存放对象的，不建议用来缓存一些有状态的对象（如长连接等）或数据等持久存储对象，因为 Pool 中的内容是会随着 GC 而被回收。

源码的开头介绍：

> An appropriate use of a Pool is to manage a group of temporary items
> silently shared among and potentially reused by concurrent independent
> clients of a package. Pool provides a way to amortize allocation overhead
> across many clients.

`sync.Pool` 适合于管理需要重复使用的对象，这些对象由 golang 的垃圾回收 GC 管理，无需调用者介入。当 GC 需要将其中部分对象进行回收时，不会告知使用者。使用者也无从得知当前 Pool 中管理有多少对象，并且每次 `Put` 和 `Get` 操作都是随机获得的对象。

本文以go1.16为例，分析下`sync.Pool`的源码实现

##  0x01    源码分析




##  0x0 参考
-   [深度分析 Golang sync.Pool 底层原理](https://www.cyhone.com/articles/think-in-sync-pool/)