---
layout: post
title: 基于golang的分布式的文件系统实现分析：bfs（计划中）
subtitle: 分析一款小文件存储系统
date: 2022-10-01
author: pandaychen
catalog: true
tags:
  - Golang
  - 分布式
---

## 0x00 前言
[bfs](https://github.com/Terry-Mao/bfs) 是基于facebook haystack 用golang实现的小文件存储系统。BFS是一个分布式图片存储系统，针对小文件、不可变进行优化。核心模块（可以横向扩展）包含如下：
- 前端接入模块proxy
- 元数据管理directory
- 单机图片存储引擎store

![arch](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/bfs/bfs-arch.png)


## 0x0 参考
-	[1.4 go在数据存储上面的应用—毛剑](https://www.slideshare.net/ssuserbefd12/14-go)
-	[BFS图片存储系统源码分析](https://scottzzq.gitbooks.io/bfs-source/content/)
- [分布式文件系统FastDFS详解](http://www.ityouknow.com/fastdfs/2018/01/06/distributed-file-system-fastdfs.html)