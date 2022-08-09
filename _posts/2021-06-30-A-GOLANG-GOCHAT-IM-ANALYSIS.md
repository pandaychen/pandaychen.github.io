---
layout:     post
title:      一个基于 golang 的轻量级 IM 项目分析：gochat
subtitle:   分析一款实时通信 IM 项目 gochat
date:       2021-06-30
author:     pandaychen
catalog:    true
tags:
    - IM
---

##  0x00    开篇
[gochat](https://github.com/LockGit/gochat) 是一款基于 golang 实现轻量级的 im 系统。技术上各层之间通过 RPC 通讯，使用 Redis 作为消息存储与投递的队列，模块间基于 etcd 的服务发现。其架构和 goim 很相像。本文简单了解下下面两个功能的实现：
-   点对点的消息发送（用户对用户）
-   点对面的消息发送（用户在房间里广播消息）

涉及到业务流程：
-   用户注册&& 用户登录
-   用户发送消息
-   用户退出登录

gochat的架构图如下：
![gochat](https://raw.githubusercontent.com/LockGit/gochat/master/architecture/gochat.png)

层级结构如下：
-   对外接口层，主要包含API模块和connect模块，API模块主要给用户提供发送消息（指定人、广播等）的接口，connect模块主要用于长连接的消息推送（注意：用户主动发送消息是不直接通过connect模块的，该模块仅为了生成长连接推送消息用）
-   内部业务逻辑：
-   消息队列：
-   内部业务Task：独立工作进程

核心模块：
-   api模块
-   connect模块
-   logic模块：处理业务逻辑，分发用户的msg
-   task模块


##	0x01	流程
笔者整理了下gochat的模块调用架构：
![architecture](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/go-chat/gochat.png)

####    用户发送信息流程
![send-message](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/gochat/send-broadcast-message-flow.png)

##	0x02	数据流程

##  0x04	参考
-	[gochat](https://github.com/LockGit/gochat)
-	[gochat 源码解析](https://blog.csdn.net/zhanglehes/article/details/115676339)