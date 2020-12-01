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


##  参考
-   [Zinx project](https://github.com/aceld/zinx/blob/master/znet/server.go)
-   [Zinx--Golang轻量级并发服务器框架](https://aceld.gitbooks.io/zinx/content/)
-   [聊聊 Go Socket 框架 Teleport 的设计思路](https://my.oschina.net/henrylee2cn/blog/2209264)
-   [gnet: 一个轻量级且高性能的 Go 网络框架](https://strikefreedom.top/go-event-loop-networking-library-gnet)