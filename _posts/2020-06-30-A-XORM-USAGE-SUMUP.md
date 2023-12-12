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
    - MySQL
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
参考[官方文档](https://gobook.io/read/gitea.com/xorm/manual-zh-CN/)

####    在同一会话（Session）执行
可以使用xorm 的会话机制（Session）来批量执行SQL，`Session.Exec` 用于执行原始 SQL 命令，并返回受影响的行数以及在操作过程中遇到的错误，如下：

```golang
session := xorm.NewSession()
defer session.Close()

sqlStmt := "INSERT INTO users (name, age) VALUES ('userA', 25)"
affected, err := session.Exec(sqlStmt)
if err != nil {
    log.Fatal(err)
}
fmt.Printf("Inserted %d rows\n", affected)

//more SQLS
sqlStmt = "UPDATE users SET age = ? WHERE name = ?"
affected, err = session.Exec(sqlStmt, 30, "userA")
if err != nil {
    log.Fatal(err)
}
fmt.Printf("Updated %d rows\n", affected)
```


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
这里采用 `xorm.io/xorm 1.0.3` 版本来实现 xorm 的 tracing 功能，支持以 Hook 钩子方式侵入 XORM 执行过程。用户侧仅需要实现 [contexts.Hook](https://gitea.com/xorm/xorm/src/branch/master/contexts/hook.go#L43) 的方法，加入 tracing 的机制即可

####    xorm 的 hooker 实现
xorm 的 `Engine` 提供了 [AddHook](https://gitea.com/xorm/xorm/src/tag/v1.3.0/engine.go#L1380) 方法，用于注入自定义钩子：
```golang
// AddHook adds a context Hook
func (engine *Engine) AddHook(hook contexts.Hook) {
    // 调用 `core.DB` 的 AddHook 方法
	engine.db.AddHook(hook)
}

// AddHook adds hook
func (db *DB) AddHook(h ...contexts.Hook) {
	db.hooks.AddHook(h...)
}
```

上述 `db.hooks.AddHook` 为 `contexts.Hooks`[类型](https://gitea.com/xorm/xorm/src/branch/master/contexts/hook.go#L14) 暴露的注册方法，传入参数为 `contexts.Hook` 类型（接口类型）：
```golang
type Hook interface {
	BeforeProcess(c *ContextHook) (context.Context, error)
	AfterProcess(c *ContextHook) error
}

type Hooks struct {
	hooks []Hook
}

func (h *Hooks) AddHook(hooks ...Hook) {
	h.hooks = append(h.hooks, hooks...)
}
```
由此了解到，如果要实现 xorm hook，需要传入一个 `contexts.Hook`，需要实现两个方法（`BeforeProcess` 和 `AfterProcess`）就能实现这个接口。

####    ContextHook
`BeforeProcess` 方法的参数是 `ContextHook`，如下，其中的参数会用于 tracing 逻辑：
```golang
// ContextHook represents a hook context
type ContextHook struct {
	start       time.Time
	Ctx         context.Context
	SQL         string        // log content or SQL
	Args        []interface{} // if it's a SQL, it's the arguments
	Result      sql.Result
	ExecuteTime time.Duration
	Err         error // SQL executed error
}
```

-   `Ctx`：本次 orm 操作的上下文，我们的 span 需要存储在这里
-   `SQL` 和 `Args`：可作为 spanLog
-   `Err`：错误，可以作为 spanLog

####    一些细节
以 `db.beforeProcess` 的实现为例（代码如下），就是实际 SQL 查询过程中调用日志和 Hook 的过程， `Hook` 参数传递使用的是指针，即将 `contexts.ContextHook` 的指针传入钩子函数执行流程，允许我们直接操作其成员 `c.Ctx` 以达到Hook的过程。

```golang
func (db *DB) beforeProcess(c *contexts.ContextHook) (context.Context, error) {
	if db.NeedLogSQL(c.Ctx) {
	    // <-- 重要，这里是将日志上下文转化成值传递
	    // 所以不能修改 context.Context 的内容
		db.Logger.BeforeSQL(log.LogContext(*c))
	}
	// Hook 是指针传递，所以可以修改 context.Context 的内容
	ctx, err := db.hooks.BeforeProcess(c)
	if err != nil {
		return nil, err
	}
	return ctx, nil
}

func (db *DB) afterProcess(c *contexts.ContextHook) error {
    // 和 beforeProcess 同理，日志上下文不能修改 context.Context 的内容
    // 而 hook 可以
	err := db.hooks.AfterProcess(c)
	if db.NeedLogSQL(c.Ctx) {
		db.Logger.AfterSQL(log.LogContext(*c))
	}
	return err
}
```


####    Hook 实现
实现代码在 [此](https://github.com/pandaychen/grpc-wrapper-framework/blob/master/storage/database/xorm/hook.go)。核心步骤三点：<br>
1、定义 `XormHook` 结构，注意不要使用该结构来进行 span 传递<br>
```golang
type XormHook struct {
	name string
}
```

2、实现 `hook` 的公共接口 <br>
```golang
// 前置钩子实现
func (h *XormHook) BeforeProcess(ctx *contexts.ContextHook) (context.Context, error) {
	span, _ := opentracing.StartSpanFromContext(ctx.Ctx, "xorm-hook")

	// 将 span 注入 c.Ctx 中
	ctx.Ctx = context.WithValue(ctx.Ctx, xormHookSpanCtxKey, span)

	return ctx.Ctx, nil
}

func (h *XormHook) AfterProcess(c *contexts.ContextHook) error {
	sp, ok := c.Ctx.Value(xormHookSpanCtxKey).(opentracing.Span)
	if !ok {
		//no span,logger?
		return nil
	}
	// 结束前上报
	defer sp.Finish()

	//log details
	if c.Err != nil {
		//log error
		sp.LogFields(tlog.Object("err", c.Err))
	}

	// 使用 xorm 的 builder 将查询语句和参数结合
	sql, err := builder.ConvertToBoundSQL(c.SQL, c.Args)
	if err == nil {
		// mark sql
		sp.LogFields(tlog.String(enums.TagDBStatement, sql))
	}
	sp.LogFields(tlog.String(enums.TagDBInstance, h.name))
	sp.LogFields(tlog.Object("args", c.Args))
	sp.SetTag(enums.TagDBExecuteCosts, c.ExecuteTime)

	return nil
}
```

3、在 xorm 包初始化时 [挂载](https://github.com/pandaychen/grpc-wrapper-framework/blob/master/storage/database/xorm/xorm.go#L49)`XormHook`<br>
```golang
func NewXormClient(option *XormOption) (*XormClient, error) {
    //...
	xormCli.Engine, err = xorm.NewEngine(option.Driver, option.Dsn)
	if err != nil {
		return nil, err
	}

	xormCli.SetDefaultContext(context.WithValue(context.Background(), clientInstance, &xormCli))
    // 注入钩子实现
	xormCli.AddHook(NewXormHook(option.Name))
    //...
	return &xormCli, nil
}
```

4、在调用时，传入上下文 `context` 至 xorm 的 `Engine.Context(ctx)`，然后运行 SQL 即可<br>


##  0x07    参考
-   [基于 Xorm 框架实现分表](https://blog.csdn.net/wyhstars/article/details/80609652)
-   [Xorm 操作指南](https://www.kancloud.cn/kancloud/xorm-manual-zh-cn)
-   [xorm - 课时 2：高级用法讲解](https://github.com/unknwon/wuwen.org/issues/6)
-   [Golang XORM 分布式链路追踪（源码分析）](https://gitee.com/avtion/xormWithTracing)