---
layout:     post
title:      Golang 中的GC小结（未完待续）
subtitle:   
date:       2022-12-01
author:     pandaychen
catalog:    true
tags:
    - GC
    - Golang
---


##  0x00    前言
作为Google的得意之作，Golang自然也用上了tcmalloc的内存池技术。因此我们普通使用Golang时，无需关注内存分配的性能问题。

##  0x01    Golang中的Map

####    为什么不用sync.Map
map并不是线程安全的。多个协程同步更新map时，会有概率导致程序core掉




##  0x0 参考