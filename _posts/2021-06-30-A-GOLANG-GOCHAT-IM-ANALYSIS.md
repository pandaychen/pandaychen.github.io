---
layout:     post
title:      一个基于 golang 的轻量级 IM 项目分析：gochat
subtitle:   分析一款实时通信 IM 项目 gochat
date:       2022-06-30
author:     pandaychen
catalog:    true
tags:
    - IM
---

##  0x00    开篇
[gochat](https://github.com/LockGit/gochat) 是一款基于 golang 实现轻量级的 im 系统。技术上各层之间通过 RPC 通讯，使用 Redis 作为消息存储与投递的队列，模块间基于 etcd 的服务发现。其架构和 goim 很相像。

![gochat](https://raw.githubusercontent.com/LockGit/gochat/master/architecture/gochat.png)

##	0x01	代码目录

##	0x02	数据流程

##  0x04	参考
-	[gochat](https://github.com/LockGit/gochat)
-	[gochat 源码解析](https://blog.csdn.net/zhanglehes/article/details/115676339)