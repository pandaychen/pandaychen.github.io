---
layout:     post
title:      Golang TCP 并发服务器框架：Zinx
subtitle:   一个基于 Golang 轻量级 TCP 并发服务器框架分析
date:       2020-08-31
author:     pandaychen
header-img: img/golang-horse-fly.png
catalog: true
category:   false
tags:
    - Zinx
    - Golang
    - 网络框架
---

##  0x00    前言
zinx的[开发文档](https://aceld.gitbooks.io/zinx/content/)基本上描述的很详细了，对初学者比较友好。笔者先前基于reactor模型实现过一个TCP网络框架[tcpframe](https://github.com/pandaychen/tcpframe)，要点是：
1.  多进程模型
2.  Reactor反应堆模式，基于事件驱动的循环
3.  基于TLV的协议通信，避免粘包

本文分析下zinx网络框架的实现的核心思路。

##  0x01    zinx的架构

##  0x02    代码分析


##  0x03    数据流程


##  0x04    细节&&总结

##  0x05    参考
-   [Zinx project](https://github.com/aceld/zinx/blob/master/znet/server.go)
-   [Zinx--Golang轻量级并发服务器框架](https://aceld.gitbooks.io/zinx/content/)
-   [聊聊 Go Socket 框架 Teleport 的设计思路](https://my.oschina.net/henrylee2cn/blog/2209264)
-   [gnet: 一个轻量级且高性能的 Go 网络框架](https://strikefreedom.top/go-event-loop-networking-library-gnet)