---
layout:     post
title:      Golang 中有趣的后台项目汇总
subtitle:   我们可以从开源项目中学习到什么？
date:       2019-12-07
author:     pandaychen
catalog:    true
header-img: img/panda-md-pic7.jpg
tags:
    - Golang
---


##	SSH 相关
-   [sshesame](https://github.com/jaksi/sshesame)，官方介绍：A fake SSH server that lets everyone in and logs their activity，可以用于实现 ssh 蜜罐
-	[cashier](https://github.com/nsheridan/cashier)，A self-service CA for OpenSSH

##	中间件

-   Google 为 Facebook 写的一个高性能 mysql 框架 [vitess](https://github.com/vitessio/vitess)，里面不少后台组件可供借鉴

-   go-redis 队列：[goworker](https://github.com/benmanns/goworker)，goworker is a Go-based background worker that runs 10 to 100,000* times faster than Ruby-based workers，高性能的消息队列，官网： https://www.goworker.org
	-	goworker 中，使用了 vitess 中的 [pool](https://github.com/vitessio/vitess/blob/master/go/pools/resource_pool.go) 实现了 redis 的连接池功能，interesting，代码在此：https://github.com/benmanns/goworker/blob/master/redis.go
	-	另外，goworker 中有一个非常经典的 workpool 实现

-   [gores](https://github.com/wang502/gores)，用 Go 语言编写的基于 Redis 的消息队列系统。核心使用 Redis 的 List 组件来模拟队列


##	认证、安全
-	[JWT-Go](https://github.com/dgrijalva/jwt-go)

##	微服务框架
-	[kratos](https://github.com/bilibili/kratos)，Kratos 是 bilibili 开源的一套 Go 微服务框架，包含大量微服务相关框架及工具。gRPC 的封装，很实用
-	[kite](https://github.com/koding/kite)，Kite is a framework for developing micro-services in Go，官方介绍：https://blog.gopheracademy.com/birthday-bash-2014/kite-microservice-library/

##	gRPC
-	[lile](https://github.com/lileio/lile)，Lile is a application generator (think create-react-app, rails new or django startproject) for gRPC services in Go and a set of tools/libraries.
-	[awesome-grpc](https://github.com/grpc-ecosystem/awesome-grpc)，A curated list of useful resources for gRPC

##	gRPC 组件
-	[gRPC+K8S-resolver](https://github.com/sercand/kuberesolver)
-	[kuberesolver](https://github.com/everflow-io/kuberesolver)

##	网关 && HTTP && Tcp-Frame
-	[Manba -  HTTP API Gateway](https://github.com/fagongzi/manba)
-	[vulcand - Programmatic load balancer backed by Etcd](https://github.com/vulcand/vulcand)
-	[fasthttp - Fast HTTP package for Go.](https://github.com/valyala/fasthttp)
-	[gnet - gnet is a high-performance, lightweight, non-blocking, event-driven networking framework written in pure Go.](https://github.com/panjf2000/gnet)

##	通用
-	[fish-shell](https://github.com/fish-shell/fish-shell)，The user-friendly command line shell.


##	其他
-	[structs](https://github.com/fatih/structs)，该库提供了比较丰富的函数，让我们可以像 python 中一样轻松的获取所有的 key 值（如 `structs.Names(server)`），所有的 value 值（如 `structs.Values(server)`），甚至直接进行类型判断（如 `structs.IsZero(server)`）等等。

##	参考
-	https://www.cnblogs.com/davygeek/p/4634919.html


转载请注明出处，本文采用 [CC4.0](http://creativecommons.org/licenses/by-nc-nd/4.0/) 协议授权
