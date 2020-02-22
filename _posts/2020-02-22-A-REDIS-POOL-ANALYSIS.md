---
layout:     post
title:      GoRedis连接池（Pool）源码分析
subtitle:   
date:       2020-02-22
author:     pandaychen
header-img:
catalog:	true
category:	false
tags:
	- Redis
	- 连接池
---

##  0x00    连接池介绍
&emsp;&emsp;连接池技术，一般是客户端侧高效管理和复用连接，避免重复创建（带来的性能损耗，特别是`TLS`）和销毁连接的一种技术手段。在项目中灵活使用连接池，对降低服务器负载十分有帮助。如`go-xorm`的[连接池](https://github.com/go-xorm/manual-zh-CN/blob/master/chapter-01/1.engine.md)、`go-redis`的[连接池](https://github.com/go-redis/redis/tree/master/internal/pool)，本文就来分析下 `go-redis` 中的连接池实现。

总览下连接池的[核心代码结构](https://github.com/go-redis/redis/blob/master/internal/pool/pool.go)，`go-redis`的连接池实现分为如下几个部分：
1.	连接池初始化、管理连接
2.	建立与关闭连接
3.	获取与放回连接
4.	监控统计 && 连接保活配置

##  0x01    结构体
[连接池选项](https://github.com/go-redis/redis/blob/master/internal/pool/pool.go#L51)定义：
```GOLANG
type Options struct {
	Dialer  func(context.Context) (net.Conn, error)
	OnClose func(*Conn) error

	PoolSize           int          //连接池大小
	MinIdleConns       int
	MaxConnAge         time.Duration    
	PoolTimeout        time.Duration    
	IdleTimeout        time.Duration
	IdleCheckFrequency time.Duration
}
```

[单个连接](https://github.com/go-redis/redis/blob/master/internal/pool/conn.go)的结构体定义：
```GOLANG
type Conn struct {
    netConn net.Conn  // tcp连接

    rd *proto.Reader // 根据 Redis 通信协议实现的 Reader
    wr *proto.Writer // 根据 Redis 通信协议实现的 Writer

    Inited    bool // 是否完成初始化
    pooled    bool // 是否放进连接池
    createdAt time.Time // 创建时间
    usedAt    int64 // 使用时间
}
```


##	参考
-   [Go Redis连接池实现](https://jackckr.github.io/golang/go-redis%E8%BF%9E%E6%8E%A5%E6%B1%A0%E5%AE%9E%E7%8E%B0/)
