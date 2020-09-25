---
layout:     post
title:      Kratos 源码分析：ORM 之 Mysql 的封装
subtitle:   分析 Kratos 的数据库 MYSQL-API
date:       2020-07-10
header-img: img/golang-tools-fun.png
author:     pandaychen
catalog:    true
tags:
    - Kratos
---

## 0x00	前言
本篇文章来分析下 Kratos 中的 Mysql 接口封装，主要是对 [`go-sql-driver`](https://github.com/go-sql-driver/mysql) 此库的封装。在其中加入了 Metrics 统计、Tracing 及熔断机制嵌入的实现。思路非常的清晰，代码主要集中在 [`sql.go`](https://github.com/go-kratos/kratos/blob/master/pkg/database/sql/sql.go) 以及 [`mysql.go`](https://github.com/go-kratos/kratos/blob/master/pkg/database/sql/mysql.go) 中。

##	0x01	go-sql-driver/mysql 基础使用
`database/sql` 与 `go-sql-driver` 两者之间的关系，golang 的 `database/sql` 源码为数据库提供了一种抽象能力，然后让第三方库比如 `go-sql-driver` 去实现。

一个基础的查询例子如下：

1.	调用 `db.Query` 执行 SQL 语句，此方法会返回一个 Rows 作为查询的结果。如果方法包含 `Query`，那么这个方法是用于查询并返回 rows 的。其他情况应该用 `Exec()`
2.	通过 `rows.Next()` 迭代查询数据
3.	通过 `rows.Scan()` 读取每一行的值，使用 `rows.Next()` 遍历结果行，使用 `rows.Scan()` 获取每行字段的值
4.	调用 `db.Close()` 关闭查询

```golang
var (
    id int
    name string
)
rows, err := db.Query("select id, name from users where id = ?", 1)
if err != nil {
    log.Fatal(err)
}
defer rows.Close()
for rows.Next() {
    err := rows.Scan(&id, &name)
    if err != nil {
        log.Fatal(err)
    }
    log.Println(id, name)
}
err = rows.Err()
if err != nil {
    log.Fatal(err)
}
```


##	0x02	Kratos 的 Mysql 配置
Kratos 的 Mysql 的配置结构 [定义如下](https://github.com/go-kratos/kratos/blob/master/pkg/database/sql/mysql.go)，定义了 master、slave 的 addr 信息、熔断器的配置以及连接地址 addr、连接池的闲置连接数 idle、最大连接数 active 以及各类超时。：

```golang
type Config struct {
	DSN          string          // write data source name.
	ReadDSN      []string        // read data source name.
	Active       int             // pool
	Idle         int             // pool
	IdleTimeout  time.Duration   // connect max life time.
	QueryTimeout time.Duration   // query sql timeout
	ExecTimeout  time.Duration   // execute sql timeout
	TranTimeout  time.Duration   // transaction sql timeout
	Breaker      *breaker.Config // breaker
}
```

关于 `Config` 配置的实例，如果配置了 readDSN，在进行读操作的时候会优先使用 readDSN 的连接。注意 readDSN 是一个数组：
```bash
[demo]
	addr = "127.0.0.1:3306"
	dsn = "{user}:{password}@tcp(127.0.0.1:3306)/{database}?timeout=1s&readTimeout=1s&writeTimeout=1s&parseTime=true&loc=Local&charset=utf8mb4,utf8"
	readDSN = ["{user}:{password}@tcp(127.0.0.2:3306)/{database}?timeout=1s&readTimeout=1s&writeTimeout=1s&parseTime=true&loc=Local&charset=utf8mb4,utf8","{user}:{password}@tcp(127.0.0.3:3306)/{database}?timeout=1s&readTimeout=1s&writeTimeout=1s&parseTime=true&loc=Local&charset=utf8,utf8mb4"]
	active = 20
	idle = 10
	idleTimeout ="4h"
	queryTimeout = "200ms"
	execTimeout = "300ms"
	tranTimeout = "400ms"
```

##	0x03	Kratos 的封装
Kratos 的封装主要针对了如下的原生的常用方法：
```golang
import "database/sql"
import _ "github.com/go-sql-driver/mysql"
```

1、连接数据库
```golang
// 来源："database/sql"
// 原型：func Open(driverName, dataSourceName string) (*DB, error)
db, err := sql.Open("mysql", "root:@tcp(127.0.0.1:3306)/test?charset=utf8")
```

2、执行写入操作`Exec`方法
```golang
result, err := db.Exec(
    "INSERT INTO users (name, age) VALUES (?, ?)",
    "gopher",
    27,
)
```

3、查询`Query`方法
```golang
rows, err := db.Query("SELECT name FROM users WHERE age = ?", age)
if err != nil {
    log.Fatal(err)
}
//调用rows.Next()遍历结果每一行
for rows.Next() {
    var name string
    if err := rows.Scan(&name); err != nil {
        log.Fatal(err)
    }
    fmt.Printf("%s is %d\n", name, age)
}
if err := rows.Err(); err != nil {
    log.Fatal(err)
}
```

4、查询单行`QueryRow`
```golang
var age int64
row := db.QueryRow("SELECT age FROM users WHERE name = ?", name)
err := row.Scan(&age)
```

5、Prepared statements
```golang
age := 27
stmt, err := db.Prepare("SELECT name FROM users WHERE age = ?")
if err != nil {
    log.Fatal(err)
}
rows, err := stmt.Query(age)
// process rows
```

6、事务操作


##	0x03	理解 Kratos 的封装
关于 Kratos 的 Mysql 封装，可以按照如下几点着手分析：
1.	`conn` 封装了原生库的 `sql.DB` 的接口（`conn` 的方法均为调用 `sql.DB` 的方法）
2.	`DB` 封装了 `conn`，有点像 `conn` 的集合，使得 `DB` 表现为一个可配置的 MYSQL 主从集群
3.	`DB` 中封装了对外的接口（方法），大部分方法都是直接调用 `conn` 封装的方法
4.	熔断器封装的层次、Tracing 封装的层次、Metrics 封装的层次（和哪个属性相关联？`conn` 还是 `row`）
5.	Kratos 中的 db 封装逻辑的执行顺序，如操作耗时，超时传递以及熔断判定及上报等

##	0x04	核心数据结构

####	sql.DB
[`sql.DB`](https://golang.org/src/database/sql/sql.go?s=21595:21652#L402) 的结构定义在此。在 Golang 中访问数据库需要用到 `sql.DB` 接口：它可以创建语句 (Statement) 和事务 (Transaction)，执行查询，获取结果。
`sql.DB` 并不是数据库连接，也并未在概念上映射到特定的数据库 (Database) 或模式 (Schema)。它只是一个抽象的接口，不同的具体驱动有着不同的实现方式。

通常而言，`sql.DB` 会处理一些重要而麻烦的事情，例如操作具体的驱动打开 / 关闭实际底层数据库的连接，按需管理连接池。`sql.DB` 这一抽象让用户不必考虑如何管理并发访问底层数据库的问题。当一个连接在执行任务时会被标记为正在使用。用完之后会放回连接池中。不过用户如果用完连接后忘记释放，就会产生大量的连接，极可能导致资源耗尽（建立太多连接，打开太多文件，缺少可用网络端口）。

```golang
type DB struct {
	// Atomic access only. At top of struct to prevent mis-alignment
	// on 32-bit platforms. Of type time.Duration.
	waitDuration int64 // Total time waited for new connections.

	connector driver.Connector
	// numClosed is an atomic counter which represents a total number of
	// closed connections. Stmt.openStmt checks it before cleaning closed
	// connections in Stmt.css.
	numClosed uint64

	mu           sync.Mutex // protects following fields
	freeConn     []*driverConn
	connRequests map[uint64]chan connRequest
	nextRequest  uint64 // Next key to use in connRequests.
	numOpen      int    // number of opened and pending open connections
	// Used to signal the need for new connections
	// a goroutine running connectionOpener() reads on this chan and
	// maybeOpenNewConnections sends on the chan (one send per needed connection)
	// It is closed during db.Close(). The close tells the connectionOpener
	// goroutine to exit.
	openerCh          chan struct{}
	resetterCh        chan *driverConn
	closed            bool
	dep               map[finalCloser]depSet
	lastPut           map[*driverConn]string // stacktrace of last conn's put; debug only
	maxIdle           int                    // zero means defaultMaxIdleConns; negative means 0
	maxOpen           int                    // <= 0 means unlimited
	maxLifetime       time.Duration          // maximum amount of time a connection may be reused
	cleanerCh         chan struct{}
	waitCount         int64 // Total number of connections waited for.
	maxIdleClosed     int64 // Total number of connections closed due to idle.
	maxLifetimeClosed int64 // Total number of connections closed due to max free limit.

	stop func() // stop cancels the connection opener and the session resetter.
}

```

Kratos 中，对 `sql.DB` 封装在 `conn` 结构中，`conn` 是基础结构，每一个 `conn` 代表一个 Mysql 的长连接，同时关联一个熔断器 breaker：
```golang
// conn database connection
type conn struct {
	*sql.DB				// 继承了 sql.DB
	breaker breaker.Breaker	// 熔断器
	conf    *Config			//MYSQL 配置
	addr    string			// conn 对应的服务端地址
}
```

而 Kratos 的 `DB` 结构对 `conn` 结构进行了封装，使得 `DB` 看起来是 One Writer,multiple Reader 的集群化结构：
```golang
// DB database.
type DB struct {
	write  *conn		// 一写
	read   []*conn		// 多读
	idx    int64
	master *DB			// 链表结构
}
```

对 `sql.Row` 的封装，结构中多加了一个 `trace.Trace` 的成员，用于 Opentracing：
```golang
// Row row.
type Row struct {
	err error
	*sql.Row			// 原生的 sql.Row
	db     *conn		// 指向哪个 conn
	query  string
	args   []interface{}
	t      trace.Trace
	cancel func()
}
```

其中 `sql.Row` 代表了 Query 的一行数据：
```golang
// Row is the result of calling QueryRow to select a single row.
type Row struct {
	// One of these two will be non-nil:
	err  error // deferred error for easy chaining
	rows *Rows
}
```

####	Stmt 结构
`Stmt` 封装了
```golang
// Stmt prepared stmt.
type Stmt struct {
	db    *conn
	tx    bool
	query string
	stmt  atomic.Value
	t     trace.Trace
}
```

##	0x05	公共方法

##
```GOLANG
func slowLog(statement string, now time.Time) {
	du := time.Since(now)
	if du > _slowLogDuration {
		log.Warn("%s slow log statement: %s time: %v", _family, statement, du)
	}
}
```

####	接口
`Open` 方法根据配置，初始化 Mysql 连接及属性，初始化 Mysql 关联的熔断器（组）：

```golang
// Open opens a database specified by its database driver name and a
// driver-specific data source name, usually consisting of at least a database
// name and connection information.
func Open(c *Config) (*DB, error) {
	db := new(DB)
	d, err := connect(c, c.DSN)
	if err != nil {
		return nil, err
	}
	addr := parseDSNAddr(c.DSN)
	// 初始化熔断组
	brkGroup := breaker.NewGroup(c.Breaker)
	brk := brkGroup.Get(addr)
	w := &conn{DB: d, breaker: brk, conf: c, addr: addr}
	rs := make([]*conn, 0, len(c.ReadDSN))
	for _, rd := range c.ReadDSN {
		d, err := connect(c, rd)
		if err != nil {
			return nil, err
		}
		addr = parseDSNAddr(rd)
		// 使用 DB 的地址作为熔断器组的 key
		brk := brkGroup.Get(addr)
		r := &conn{DB: d, breaker: brk, conf: c, addr: addr}
		rs = append(rs, r)
	}
	db.write = w
	db.read = rs
	db.master = &DB{write: db.write}
	return db, nil
}
```

上面代码中，`Open` 方法中调用 `connect` 方法打开（创建）一个 SQL 连接，这里是调用了原生的 `sql.Open` 方法来新建，同时设置 mysql 的连接属性：
```golang
func connect(c *Config, dataSourceName string) (*sql.DB, error) {
	d, err := sql.Open("mysql", dataSourceName)
	if err != nil {
		err = errors.WithStack(err)
		return nil, err
	}
	d.SetMaxOpenConns(c.Active)
	d.SetMaxIdleConns(c.Idle)
	d.SetConnMaxLifetime(time.Duration(c.IdleTimeout))
	return d, nil
}
```

##	0x06	DB 结构及封装
真正对外暴露的接口是 `DB` 及其封装的方法，[`DB` 结构](https://github.com/go-kratos/kratos/blob/master/pkg/database/sql/sql.go#L38) 如下，从结构看，Kratos 的 Mysql 结构可以定义为一主多从的 Mysql 集群方式。
```golang
// DB database.
type DB struct {
	write  *conn		// 一写
	read   []*conn		// 多读
	idx    int64
	master *DB
}
```
`DB` 结构提供了如下对外方法：
-	`Begin` 方法：调用 `write` 作为 master 连接开启一个事务

```golang
// Begin starts a transaction. The isolation level is dependent on the driver.
func (db *DB) Begin(c context.Context) (tx *Tx, err error) {
	return db.write.begin(c)
}
```

2、`Exec` 方法，执行写入数据，实际封装了调用 `db.write.exec`：

```golang
// Exec executes a query without returning any rows.
// The args are for any placeholder parameters in the query.
func (db *DB) Exec(c context.Context, query string, args ...interface{}) (res sql.Result, err error) {
	return db.write.exec(c, query, args...)
}
```

-

```golang
// Prepare creates a prepared statement for later queries or executions.
// Multiple queries or executions may be run concurrently from the returned
// statement. The caller must call the statement's Close method when the
// statement is no longer needed.
func (db *DB) Prepare(query string) (*Stmt, error) {
	return db.write.prepare(query)
}
```

-

```golang
// Query executes a query that returns rows, typically a SELECT. The args are
// for any placeholder parameters in the query.
func (db *DB) Query(c context.Context, query string, args ...interface{}) (rows *Rows, err error) {
	idx := db.readIndex()
	for i := range db.read {
		if rows, err = db.read[(idx+i)%len(db.read)].query(c, query, args...); !ecode.EqualError(ecode.ServiceUnavailable, err) {
			return
		}
	}
	return db.write.query(c, query, args...)
}
```


##	conn 封装的方法
`conn` 的 `exec` 方法的上层方法是 `db.Exec`，[](https://golang.org/pkg/database/sql/#DB.ExecContext)，实现代码如下：
```golang
func (db *conn) exec(c context.Context, query string, args ...interface{}) (res sql.Result, err error) {
	now := time.Now()
	defer slowLog(fmt.Sprintf("Exec query(%s) args(%+v)", query, args), now)
	if t, ok := trace.FromContext(c); ok {
		t = t.Fork(_family, "exec")
		t.SetTag(trace.String(trace.TagAddress, db.addr), trace.String(trace.TagComment, query))
		defer t.Finish(&err)
	}
	if err = db.breaker.Allow(); err != nil {
		_metricReqErr.Inc(db.addr, db.addr, "exec", "breaker")
		return
	}
	_, c, cancel := db.conf.ExecTimeout.Shrink(c)
	res, err = db.ExecContext(c, query, args...)
	cancel()
	db.onBreaker(&err)
	_metricReqDur.Observe(int64(time.Since(now)/time.Millisecond), db.addr, db.addr, "exec")
	if err != nil {
		err = errors.Wrapf(err, "exec:%s, args:%+v", query, args)
	}
	return
}
```
简单梳理下 `exec` 的流程：
![image](https://wx2.sbimg.cn/2020/08/12/oxaQN.png)

```golang
// Stmt prepared stmt.
type Stmt struct {
	db    *conn
	tx    bool
	query string
	stmt  atomic.Value
	t     trace.Trace
}
```


##  0x07	Metrics 统计
Metrics 的定义的结构 [代码在此](https://github.com/go-kratos/kratos/blob/master/pkg/database/sql/metrics.go)，包含了下面几个维度：
1.	`_metricReqDur`
2.	`_metricReqErr`
3.	`_metricConnTotal`
4.	`_metricConnCurrent`

```golang
var (
	_metricReqDur = metric.NewHistogramVec(&metric.HistogramVecOpts{
		Namespace: namespace,
		Subsystem: "requests",
		Name:      "duration_ms",
		Help:      "mysql client requests duration(ms).",
		Labels:    []string{"name", "addr", "command"},
		Buckets:   []float64{5, 10, 25, 50, 100, 250, 500, 1000, 2500},
	})
	_metricReqErr = metric.NewCounterVec(&metric.CounterVecOpts{
		Namespace: namespace,
		Subsystem: "requests",
		Name:      "error_total",
		Help:      "mysql client requests error count.",
		Labels:    []string{"name", "addr", "command", "error"},
	})
	_metricConnTotal = metric.NewCounterVec(&metric.CounterVecOpts{
		Namespace: namespace,
		Subsystem: "connections",
		Name:      "total",
		Help:      "mysql client connections total count.",
		Labels:    []string{"name", "addr", "state"},
	})
	_metricConnCurrent = metric.NewGaugeVec(&metric.GaugeVecOpts{
		Namespace: namespace,
		Subsystem: "connections",
		Name:      "current",
		Help:      "mysql client connections current.",
		Labels:    []string{"name", "addr", "state"},
	})
)
```

##	0x08	Tracing 的封装


##	0x09	熔断 Breaker 机制嵌入
首先，看下在 `Row` 类似的方法 `Scan`，它封装了 `sql.Row` 的 `Scan` 方法，根据 `r.db.onBreaker` 返回的结果，使用 `	r.db.onBreaker(&err)` 进行状态上报。
数据上报到滑动窗口中进行统计。然后在 `Allow()` 方法中进行熔断状态判定。

```golang
// Scan copies the columns from the matched row into the values pointed at by dest.
func (r *Row) Scan(dest ...interface{}) (err error) {
	defer slowLog(fmt.Sprintf("Scan query(%s) args(%+v)", r.query, r.args), time.Now())
	if r.t != nil {
		defer r.t.Finish(&err)
	}
	if r.err != nil {
		err = r.err
	} else if r.Row == nil {
		err = ErrStmtNil
	}
	if err != nil {
		return
	}
	err = r.Row.Scan(dest...)
	if r.cancel != nil {
		r.cancel()
	}
	// 根据 err 的结果进行熔断器数据上报
	r.db.onBreaker(&err)
	if err != ErrNoRows {
		err = errors.Wrapf(err, "query %s args %+v", r.query, r.args)
	}
	return
}
```

`onBreaker` 的方法，调用 `db.breaker.MarkFailed()` 或者 `db.breaker.MarkSuccess()` 进行熔断数据上报，注意 err 的判定类型：
```golang
func (db *conn) onBreaker(err *error) {
	if err != nil && *err != nil && *err != sql.ErrNoRows && *err != sql.ErrTxDone {
		db.breaker.MarkFailed()
	} else {
		db.breaker.MarkSuccess()
	}
}
```

##	0x0A	Row 及封装
`sql.Row` 代表了单行查询结果：Row is the result of calling QueryRow to select a single row.<br>
原生的 `sql.Row`[定义](https://golang.org/src/database/sql/sql.go?s=91483:91600#L3175) 及提供的 [两个方法](https://golang.org/pkg/database/sql/#Row)：
```golang
// Row is the result of calling QueryRow to select a single row.
type Row struct {
	// One of these two will be non-nil:
	err  error // deferred error for easy chaining
	rows *Rows
}

func (r *Row) Err() error
func (r *Row) Scan(dest ...interface{}) error
```

Kratos 封装了 `sql.Row`，结构如下，注意这里是以 `*sql.Row` 方式封装的，所以，修改需要改动的方法即可，不需要调整的方法则使用 `sql.Row` 原生的方法即可：

```golang
// Row row.
type Row struct {
	err error
	*sql.Row		// 封装了原生的 sql.Row
	db     *conn
	query  string
	args   []interface{}
	t      trace.Trace
	cancel func()
}
```

`Scan` 方法，该方法用来获取单行数据，我们先看下 [原生的实现](https://golang.org/src/database/sql/sql.go?s=91886:91931#L3186)：
```golang
func (r *Row) Scan(dest ...interface{}) error {
	if r.err != nil {
		return r.err
	}

	defer r.rows.Close()
	for _, dp := range dest {
		if _, ok := dp.(*RawBytes); ok {
			return errors.New("sql: RawBytes isn't allowed on Row.Scan")
		}
	}

	if !r.rows.Next() {
		if err := r.rows.Err(); err != nil {
			return err
		}
		return ErrNoRows
	}
	err := r.rows.Scan(dest...)
	if err != nil {
		return err
	}
	// Make sure the query can be processed to completion with no errors.
	return r.rows.Close()
}
```

再看下修改封装后的实现，对 `Scan` 方法的封装主要多了两点，（这里有疑问是为何没有进行熔断器的判定逻辑）：
1.	使用 `defer slowLog` 计算耗时
2.	使用 `r.db.onBreaker(&err)` 上传熔断器统计状态

```golang
// Scan copies the columns from the matched row into the values pointed at by dest.
func (r *Row) Scan(dest ...interface{}) (err error) {
	defer slowLog(fmt.Sprintf("Scan query(%s) args(%+v)", r.query, r.args), time.Now())
	if r.t != nil {
		defer r.t.Finish(&err)
	}
	if r.err != nil {
		err = r.err
	} else if r.Row == nil {
		err = ErrStmtNil
	}
	if err != nil {
		return
	}
	// 调用原生的 Scan 方法获取数据
	err = r.Row.Scan(dest...)
	if r.cancel != nil {
		r.cancel()
	}
	// 根据 err 上报熔断器统计状态
	r.db.onBreaker(&err)
	if err != ErrNoRows {
		err = errors.Wrapf(err, "query %s args %+v", r.query, r.args)
	}
	return
}
```

##	0x0B	总结
本文从代码层面分析了 Kratos 对 `sql.DB` 库的封装逻辑，通过本篇文章，对微服务与 Orm 的应用结合有了更加深入的认知。

##  0x0C	参考


