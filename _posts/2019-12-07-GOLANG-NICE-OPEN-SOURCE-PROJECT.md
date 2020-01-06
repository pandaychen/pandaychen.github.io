---
layout:     post
title:      Golang中有趣的后台项目汇总
subtitle:   我们可以从开源项目中学习到什么？
date:       2019-12-07
author:     pandaychen
catalog:    true
header-img: img/panda-md-pic7.jpg
tags:
    - Golang
---

##  后台项目

###	SSH相关
-   [sshesame](https://github.com/jaksi/sshesame)，官方介绍：A fake SSH server that lets everyone in and logs their activity，可以用于实现ssh蜜罐

###	后台组件

-   Google为Facebook写的一个高性能mysql框架[vitess](https://github.com/vitessio/vitess)，里面不少后台组件可供借鉴

-   go-redis队列：[goworker](https://github.com/benmanns/goworker)，goworker is a Go-based background worker that runs 10 to 100,000* times faster than Ruby-based workers，高性能的消息队列，官网： https://www.goworker.org
	-	goworker中，使用了vitess中的[pool](https://github.com/vitessio/vitess/blob/master/go/pools/resource_pool.go)实现了redis的连接池功能，interesting，代码在此：https://github.com/benmanns/goworker/blob/master/redis.go
	-	另外，goworker中有一个非常经典的workpool实现


##	认证、安全
-	[JWT-Go](https://github.com/dgrijalva/jwt-go)

##	微服务框架
-	[kratos](https://github.com/bilibili/kratos)，Kratos是bilibili开源的一套Go微服务框架，包含大量微服务相关框架及工具。gRPC的封装，很实用
-	[kite](https://github.com/koding/kite)，Kite is a framework for developing micro-services in Go，官方介绍：https://blog.gopheracademy.com/birthday-bash-2014/kite-microservice-library/

##	gRPC
-	[lile](https://github.com/lileio/lile)，Lile is a application generator (think create-react-app, rails new or django startproject) for gRPC services in Go and a set of tools/libraries.
-	[awesome-grpc](https://github.com/grpc-ecosystem/awesome-grpc)，A curated list of useful resources for gRPC

##	通用
-	[fish-shell](https://github.com/fish-shell/fish-shell)，The user-friendly command line shell. 

##	参考
-	https://www.cnblogs.com/davygeek/p/4634919.html


转载请注明出处，本文采用 [CC4.0](http://creativecommons.org/licenses/by-nc-nd/4.0/) 协议授权
