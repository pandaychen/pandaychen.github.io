---
layout: post
title: Golang 的分布式缓存：GroupCache 分析
subtitle: 
date: 2022-05-30
author: pandaychen
catalog: true
tags:
    - GroupCache
    - 缓存
---

##  0x00    前言
[groupcache](https://github.com/golang/groupcache) 是一个分布式缓存库，支持多节点互备热数据，有良好的稳定性和较高的并发性。


##  0x05  参考
-   [dl.google.com: Powered by Go](https://go.dev/talks/2013/oscon-dl.slide#1)
-   [groupcache 架构设计](https://www.jianshu.com/p/f69f3a3a9a78)