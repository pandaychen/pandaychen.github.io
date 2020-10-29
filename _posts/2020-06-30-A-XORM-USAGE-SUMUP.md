---
layout:     post
title:      Xorm 使用总结（Mysql）
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
engine, err := xorm.NewEngine("mysql", "user:pwd@tcp(ip:port)/dbname?charset=utf8")
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

##  0x02    Xorm 事务
使用 Xorm 事务处理时，需要创建 `Session`，可以混用 ORM 方法和 RAW 方法，如下（注意：Mysql 引擎为 innodb 才支持事务，myisam 是不支持事务）：
```golang
func main{
    session := engine.NewSession()
    defer session.Close()
    // add Begin() before any action
    // 开启事务
    err := session.Begin()
    user1 := Userinfo{Username: "pandaychen", Created: time.Now()}
    _, err = session.Insert(&user1)
    if err != nil {
        // 发生错误时进行回滚
        session.Rollback()
        return
    }
    user2 := Userinfo{Username: "panda"}
    _, err = session.Where("id = ?", 2).Update(&user2)
    if err != nil {
        session.Rollback()
        return
    }

    _, err = session.Exec("delete from userinfo where username = ?", user2.Username)
    if err != nil {
        session.Rollback()
        return
    }

    // add Commit() after all actions
    // 完成事务
    err = session.Commit()
    if err != nil {
        return
    }
}
```

##  0x03    Xorm 的事件钩子
Xorm 支持事件钩子，见 [文档](https://gobook.io/read/gitea.com/xorm/manual-zh-CN/chapter-12/index.html)。比如 `BeforeInsert` 和 `AfterInsert`（前者在进行插入记录之前被调用，后者在完成插入记录之后被调用）：
```golang
func (a *Account) BeforeInsert() {
	log.Printf("before insert: %s", a.Name)
}

func (a *Account) AfterInsert() {
	log.Printf("after insert: %s", a.Name)
}
```

##  0x04    Xorm 常用函数及方法

##  0x05    基于 Xorm 的自动分表
本小节简单介绍下如何利用 Xorm 实现 Mysql 分表
-   分表的优点：避免单表数据量过大带来的操作性能瓶颈
-   分表的缺点：数据的关联性可能受到影响，可能会消耗额外的查询（如按表遍历）及运算压力等
-   分表的方式：按天、按业务等

####    分表的需求
通常，创建 Mysql Engine 的时候，使用下面的 `engine` 来操作具体的数据库 Table，那么问题来了，Table 的名字如何自动切换？
```golang
engine, err := xorm.NewEngine("mysql", "user:pwd@tcp(ip:port)/dbname?charset=utf8")
```

针对分表的场景，需要满足如下需求：
1. 如何定时的自动创建一个新表（类似 Crontab）
2. 如何解决 Golang 结构去创建一个以时间结尾为表名的数据库表（固定 prefix 或者 suffix）
3. 如何解决 golang 类和表的映射问题，若映射不对，则影响数据插入操作（engine 和表的 mapping 问题）

####    自动创建新表
可以利用 `Cron` 解决定时创建新表的问题
```golang
func NewDBEngine(dburl string) (*DBEngine, error) {
    fmt.Println(dburl)
    db, err := xorm.NewEngine("mysql", dburl)
    if err != nil {
        return nil, err
    }
    err = db.Ping()
    if err != nil {
        return nil, err
    }
    db.SetMaxOpenConns(10)

    db.ShowSQL(false)
    log.Infof("connect to database(%v) server OK!", dburl)

    engine := DBEngine{
        dbengine: db,
    }
    engine.dbengine.SetMapper(core.GonicMapper{})

    engine.Cron = cron.New()
    engine.Cron.AddFunc(SPEC, engine.InitTable)
    engine.commParam = make(chan CommParam, GET_CACHE_DATA)
    Init(&engine)
    return &engine, nil
}
```

####    自定义 Table（固定 fix）
可以利用 xorm mapper 解决，xorm 支持驼峰或自定义的表格命名方式
```golang
func (db *DBEngine) InitTable() {
    var date string
    if int(time.Now().Month()) >= 10 {
        date = "_" + strconv.Itoa(int(time.Now().Year())) + strconv.Itoa(int(time.Now().Month()))
    } else {
        date = "_" + strconv.Itoa(int(time.Now().Year())) + "0" + strconv.Itoa(int(time.Now().Month()))
    }
    tbMapper := core.NewSuffixMapper(core.GonicMapper{}, date)
    db.dbengine.SetTableMapper(tbMapper)

    for key, v := range TablePrefix {
        isExist, err := db.dbengine.IsTableExist(key + date)
        if err != nil {
            log.Errorf("[Database] inittable failed for %v", err)
            return
        }
        if isExist {
            log.Infof("[Database] The %v table  is exist!!", key+date)
        } else {
            err := db.dbengine.Sync2(v)
            if err != nil {
                log.Errorf("[Database] Create table error for %v %v", err)
            }
        }
    }
}
```

####    自动映射表
利用 mapper 改变表格的命名和类的映射关系
```golang
var TablePrefix = map[string]interface{}{
    "gw_rx":     model.Gw_rx{},
    "gw_tx":     model.Gw_tx{},
    "gw_stats":  model.Gw_stats{},
    "mac_tx":    model.Mac_tx{},
    "mac_rx":    model.Mac_rx{},
    "mac_error": model.Mac_error{},
    "app_rx":    model.App_rx{},
    "app_tx":    model.App_tx{},
    "app_join":  model.App_join{},
    "app_ack":   model.App_ack{},
    "app_error": model.App_error{},
    "rxinfo":    model.Rxinfo{},
    "txinfo":    model.Txinfo{},
}
```

##  0x06    Xorm Tracing 实现

##  0x07    参考
-   [基于 Xorm 框架实现分表](https://blog.csdn.net/wyhstars/article/details/80609652)
-   [Xorm 操作指南](https://www.kancloud.cn/kancloud/xorm-manual-zh-cn)
-   [xorm - 课时 2：高级用法讲解](https://github.com/unknwon/wuwen.org/issues/6)