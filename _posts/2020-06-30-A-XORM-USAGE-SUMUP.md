---
layout:     post
title:      Xorm 使用总结
subtitle:   Xorm-Mysql 用法及避坑总结
date:       2020-06-29
author:     pandaychen
header-img:
catalog: true
category:   false
tags:
    - Xorm
---

##  0x00    前言
本文介绍 Xorm-MYSQL 的使用（自动分表）、超时封装、Tracing 封装及日常遇到的问题。Xorm 支持多种数据库驱动, 如: Mysql、Mariadb、Tidb、Postgres、Oracle 等等。

##  0x01  基础
Xorm-Mysql 的基础使用方式汇总如下，其他细节可以参见 Xorm 的 [官方文档](http://gobook.io/read/gitea.com/xorm/manual-en-US/)

1、Engine 相关 <br>
在项目中需要注意的是可以使用 goroutine 和 `engine.Ping()` 的方式来实现 Mysql 长连接保活机制。
```golang
// 创建 engine 对象
engine, err := xorm.NewEngine("mysql", "user:pwd@tcp(ip:port)/db?charset=utf8")
if err != nil {
    log.Fatalf("init engine fail! err:%+v", err)
}

// 连接池配置
engine.SetMaxOpenConns(30)                  // 最大 db 连接
engine.SetMaxIdleConns(10)                  // 最大 db 连接空闲数
engine.SetConnMaxLifetime(30 * time.Minute) // 超过空闲数连接存活时间

// 日志相关配置
engine.ShowSQL(true)                      // 打印日志
engine.Logger().SetLevel(core.LOG_DEBUG) // 打印日志级别
engine.SetLogger()                       // 设置日志输出 (控制台, 日志文件, 系统日志等)


// 测试连通性
if err = engine.Ping(); err != nil {
    log.Fatalf("ping to db fail! err:%+v", err)
}
```

##  0x02    基于Xorm的自动分表

##  参考
-   [基于 Xorm 框架实现分表](https://blog.csdn.net/wyhstars/article/details/80609652)
-   [Xorm 操作指南](https://www.kancloud.cn/kancloud/xorm-manual-zh-cn)