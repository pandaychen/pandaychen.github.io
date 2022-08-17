---
layout:     post
title:      Go-Redis 连接池（Pool）源码分析
subtitle:   分析一款典型的 redis 连接池实现
date:       2020-02-22
author:     pandaychen
header-img:
catalog: true
category:   false
tags:
    - Redis
    - 连接池
---

##  0x00    介绍
连接池技术，一般是客户端侧高效管理和复用连接，避免重复创建（带来的性能损耗，特别是 `TLS`）和销毁连接的一种技术手段。在项目中灵活使用连接池，对降低服务器负载十分有帮助；此外，在司内的 DB 托管场景，如遇后台升配、代理扩容等场景，如果服务内置了连接池，一般不需要重启（因为连接池会自动尝试重建）。如 go-xorm 的 [连接池](https://github.com/go-xorm/manual-zh-CN/blob/master/chapter-01/1.engine.md)、go-redis 的 [连接池](https://github.com/go-redis/redis/tree/master/internal/pool)，本文就来分析下 go-redis 中的连接池实现。

总览下连接池的核心 [代码结构](https://github.com/go-redis/redis/blob/master/internal/pool/pool.go)，go-redis 的连接池实现分为如下几个部分：
1.	连接池初始化、管理连接
2.	建立与关闭连接
3.	获取与放回连接，核心实现[Get](https://github.com/go-redis/redis/blob/master/internal/pool/pool.go#L244)、[Put](https://github.com/go-redis/redis/blob/master/internal/pool/pool.go#L354)
4.	监控统计 && 连接保活配置

连接池主要的思想是把新建的连接暂存到连接池中，当请求结束后不关闭连接，而是放回到连接池中，需要的时候从连接池中取出连接使用；同时，根据一定的策略（如超时）定时回收连接；此外，有些连接池还会对连接池中未失效的连接进行健康度检测，防止连接异常

####    go-redis 连接池的特点
-   使用 slice 存储（构建）连接池，每个单位代表一个连接
-   支持自动关闭（回收）空闲的连接
-   支持设置连接的最大存活时间
-   支持统计连接池的状态
-   不支持单个连接的健康检查，需要用户自行在业务层实现


##  0x01    Pool 相关的结构体

####    连接池配置选项

连接池 [选项](https://github.com/go-redis/redis/blob/master/internal/pool/pool.go#L51) 定义：
```golang
type Options struct {
	Dialer  func(context.Context) (net.Conn, error)    // 新建连接的工厂函数
	OnClose func(*Conn) error                   // 关闭连接的回调函数

	PoolSize           int          // 连接池大小
	MinIdleConns       int          // 最小空闲连接数
	MaxConnAge         time.Duration        // 连接池获取的超时时间
	PoolTimeout        time.Duration        // 空闲连接的超时时间
	IdleTimeout        time.Duration    // 空闲连接的超时时间
	IdleCheckFrequency time.Duration    // 检查空闲连接频率（超时空闲连接清理的间隔时间）
}
```

上面的选项是在初始化 go-redis 客户端时，需要合理设置。项目中一般我们做如下设置：
```golang
func InitRedisClient() *redis.Client{
    //...
    return redis.NewClient(&redis.Options{
        Addr:     redis_addr,
        Password: redis_cfg.RedisPass, // no password set
        DB:       0,                  // use default DB
        ReadTimeout:  time.Millisecond * time.Duration(500),
        WriteTimeout: time.Millisecond * time.Duration(500),
        IdleTimeout: time.Second * time.Duration(60),
        PoolSize:    64,
        MinIdleConns: 16,
    })
}
```

####    连接Conn

[单个连接](https://github.com/go-redis/redis/blob/master/internal/pool/conn.go) 的结构体定义，核心是 `net.Conn` 封装及根据 Redis 协议实现相应的读写操作：
```go
type Conn struct {
    // 包装net.conn
    netConn net.Conn  // tcp 连接

    rd *proto.Reader // 根据 Redis 通信协议实现的 Reader
    wr *proto.Writer // 根据 Redis 通信协议实现的 Writer

    Inited    bool // 是否完成初始化（该连接是否初始化，比如如果需要执行命令之前需要执行的auth,select db 等的标识，代表已经auth,select过）
    pooled    bool // 是否放进连接池
    createdAt time.Time // 创建时间（超过maxconnage的链接需要淘汰）
    usedAt    int64 // 使用时间（该链接执行命令的时候所记录的时间，就是上次用过这个链接的时间点）
}
```

####    核心结构：ConnPool
连接池 `ConnPool` 的结构体定义：
```golang
type ConnPool struct {
    opt *Options // 连接池参数配置，如上

    dialErrorsNum uint32 // 连接失败的错误次数，atomic

    lastDialErrorMu sync.RWMutex // 上一次连接错误锁，读写锁
    lastDialError   error // 上一次连接错误（保存了最近一次的连接错误）

    queue chan struct{}     // 轮转队列，是一个 channel 结构

    connsMu      sync.Mutex // 连接队列锁
    conns        []*Conn // 连接队列（连接队列，维护了未被删除所有连接）
    idleConns    []*Conn // 空闲连接队列（空闲连接队列，维护了所有的空闲连接）
    poolSize     int // 连接池大小
    idleConnsLen int // 空闲连接队列长度

    stats Stats // 连接池统计的结构体（包含了使用数据）

    _closed  uint32 // 连接池关闭标志，atomic
    closedCh chan struct{} // 通知连接池关闭通道（用于主协程通知子协程的常用方法）
}
```

连接池统计 `Stats` 的结构定义：
```golang
// Stats contains pool state information and accumulated stats.
type Stats struct {
	Hits     uint32 // number of times free connection was found in the pool
	Misses   uint32 // number of times free connection was NOT found in the pool
	Timeouts uint32 // number of times a wait timeout occurred

	TotalConns uint32 // number of total connections in the pool
	IdleConns  uint32 // number of idle connections in the pool
	StaleConns uint32 // number of stale connections removed from the pool
}
```

##  0x02    Pool 接口
首先，封装 Pool 的接口实现方法 `Pooler`，如下：
```golang
//连接池的接口
type Pooler interface {
    NewConn(context.Context) (*Conn, error) // 创建连接
    CloseConn(*Conn) error // 关闭连接

    Get(context.Context) (*Conn, error) // 获取连接
    Put(*Conn) // 放回连接
    Remove(*Conn, error) // 移除连接

    Len() int // 连接池长度
    IdleLen() int // 空闲连接数量
    Stats() *Stats // 连接池统计

    Close() error // 关闭连接池
}
```

对外接口包含4个主要模块：

-   建立连接和关闭连接
-   从池中取`Conn`的管理
-   监控统计
-   整个`Pooler`池的关闭

##  0x03    连接池管理功能

####    1、初始化（新建）连接池
使用工厂函数创建全局 `Pool` 对象，首先会初始化一个 `ConnPool` 实例，初始化 `PoolSize` 大小的连接队列和轮转队列；然后在 `checkMinIdleConns` 方法中，会根据 `MinIdleConns` 参数维持一个最小连接数，以保证连接池中有这么多数量的连接处于活跃状态。

`IdleTimeout` 和 `IdleCheckFrequency` 参数用来在每过一段时间内会对连接池中不活跃的连接做清理操作，如果同时设置了，则异步开启回收连接的 goroutine，即 `reaper` 方法；再看 `ConnPool` 结构体的定义，`conns` 和 `idleConns` 这两个队列，`conns` 就是连接队列，而 `idleConns` 就是轮转队列。两者作用如下：

-   `conns`：
-   `idleConns`：

```golang
func NewConnPool(opt *Options) *ConnPool {
    p := &ConnPool{
        opt: opt,   //Save 配置
        // 创建存储工作连接的缓冲通道
        queue:     make(chan struct{}, opt.PoolSize),
        // 创建存储所有 conn 的切片
        conns:     make([]*Conn, 0, opt.PoolSize),
        // 创建存储所有 conn 的切片
        idleConns: make([]*Conn, 0, opt.PoolSize),
        // 用于通知所有协程连接池已经关闭的通道
        closedCh:  make(chan struct{}),
    }

    // 检查连接池的空闲连接数量是否满足最小空闲连接数量要求
    // 若不满足，则创建足够的空闲连接
    p.checkMinIdleConns()

    if opt.IdleTimeout > 0 && opt.IdleCheckFrequency > 0 {
        // 如果配置了空闲连接超时时间 && 检查空闲连接频率
        // 则单独开启一个清理空闲连接的协程（定期）
        go p.reaper(opt.IdleCheckFrequency)
    }

    return p
}

// 根据 MinIdleConns 参数，初始化即创建指定数量的连接
func (p *ConnPool) checkMinIdleConns() {
	if p.opt.MinIdleConns == 0 {
		return
	}
	for p.poolSize < p.opt.PoolSize && p.idleConnsLen < p.opt.MinIdleConns {
		p.poolSize++
		p.idleConnsLen++
		go func() {
            // 异步建立 redis 连接，以满足连接池的最低要求
			err := p.addIdleConn()
			if err != nil {
                p.connsMu.Lock()
                 // 创建 redis 失败时则减去计数，注意先 Lock
				p.poolSize--
				p.idleConnsLen--
				p.connsMu.Unlock()
			}
		}()
	}
}

// 空闲连接的建立
func (p *ConnPool) addIdleConn() error {
	cn, err := p.dialConn(context.TODO(), true)
	if err != nil {
		return err
	}

	p.connsMu.Lock()
	defer p.connsMu.Unlock()

	// It is not allowed to add new connections to the closed connection pool.
	if p.closed() {
		_ = cn.Close()
		return ErrClosed
	}

	p.conns = append(p.conns, cn)       // 增加连接队列
	p.idleConns = append(p.idleConns, cn)   // 增加轮转队列
	return nil
}

```

####    2、关闭连接池
关闭连接池方法如下，主要完成释放连接资源的工作：
```golang
func (p *ConnPool) Close() error {
    // 原子性检查连接池是否已经关闭，若没关闭，则将关闭标志置为 1
    if !atomic.CompareAndSwapUint32(&p._closed, 0, 1) {
        return ErrClosed
    }

    // 关闭 closedCh 通道
    // 连接池中的所有协程都可以通过判断该通道是否关闭来确定连接池是否已经关闭
    close(p.closedCh)

    var firstErr error
    //Lock
    p.connsMu.Lock()
    // 连接队列锁上锁，关闭队列中的所有连接，并置空所有维护连接池状态的数据结构
    for _, cn := range p.conns {
        if err := p.closeConn(cn); err != nil && firstErr == nil {
            firstErr = err
        }
    }
    p.conns = nil
    p.poolSize = 0
    p.idleConns = nil
    p.idleConnsLen = 0
    p.connsMu.Unlock()

    return firstErr
}
```

下一节，分析下连接池的管理功能，如何从连接池取出和放回连接。

##  0x04    连接池的单个连接管理

通过 `dialConn` 方法生成新建的连接，注意参数 `cn.pooled`，通常情况下新建的连接会插入到连接池的 `conns` 队列中，当发现连接池的大小超出了设定的连接大小时，这时候新建的连接的 `cn.pooled` 属性被设置为 `false`，该连接未来将会被删除，不会落入连接池

####    1、建立连接
```golang
func (p *ConnPool) newConn(ctx context.Context, pooled bool) (*Conn, error) {
    // 拨号
    cn, err := p.dialConn(ctx, pooled)
    if err != nil {
        return nil, err
    }

    p.connsMu.Lock()
    p.conns = append(p.conns, cn)   //conns 保存所有新建连接
    if pooled {
        // If pool is full remove the cn on next Put.
        if p.poolSize >= p.opt.PoolSize {
            cn.pooled = false
        } else {
            p.poolSize++
        }
    }
    p.connsMu.Unlock()
    return cn, nil
}


func (p *ConnPool) dialConn(ctx context.Context, pooled bool) (*Conn, error) {
    if p.closed() {
        return nil, ErrClosed
    }

    if atomic.LoadUint32(&p.dialErrorsNum) >= uint32(p.opt.PoolSize) {
        return nil, p.getLastDialError()
    }

    netConn, err := p.opt.Dialer(ctx)
    if err != nil {
        p.setLastDialError(err)
        if atomic.AddUint32(&p.dialErrorsNum, 1) == uint32(p.opt.PoolSize) {
            go p.tryDial()
        }
        return nil, err
    }

    cn := NewConn(netConn)
    cn.pooled = pooled
    return cn, nil
}

func (p *ConnPool) tryDial() {
    for {
        if p.closed() {
            return
        }

        conn, err := p.opt.Dialer(context.Background())
        if err != nil {
            p.setLastDialError(err)
            time.Sleep(time.Second)
            continue
        }

        atomic.StoreUint32(&p.dialErrorsNum, 0)
        _ = conn.Close()
        return
    }
}
```

####    2、从连接池取出连接
在每次真正执行操作之前，client 都会调用 `ConnPool` 的 `Get` 方法，在此`Get` 方法中实现了连接的创建和获取。可以看到，**每次取出连接都是以出栈的方式取出切片里的最后一个空闲连接**。从连接池中获取一个连接的过程如下：

1、首先，检查连接池是否被关闭，如果被关闭直接返回 `ErrClosed` 错误 <br>

2、尝试在轮转队列 `queue` 中占据一个位置（即尝试申请一个令牌），如果抢占的等待时间超过连接池的超时时间，会返回 `ErrPoolTimeout` 错误，见下面的 `waitTurn` 方法 <br>

轮转队列的主要作用是协调连接池的生产 - 消费过程，即使用令牌机制保证每往轮转队列中添加一个元素时，可用的连接资源的数量就减少一。若无法立即写入，该过程将尝试等待 `PoolTimeout` 时间后，返回相应结果。

注意下面的 `timers` 基于 `sync.Pool` 做了优化。

```golang
func (p *ConnPool) waitTurn(ctx context.Context) error {
    select {
    case <-ctx.Done():
        return ctx.Err()
    default:
    }

    select {
    case p.queue <- struct{}{}:
        return nil
    default:
    }

    timer := timers.Get().(*time.Timer)
    timer.Reset(p.opt.PoolTimeout)

    select {
    case <-ctx.Done():
        if !timer.Stop() {
            <-timer.C
        }
        timers.Put(timer)
        return ctx.Err()
    case p.queue <- struct{}{}:
        if !timer.Stop() {
            <-timer.C
        }
        timers.Put(timer)
        return nil
    case <-timer.C:
        timers.Put(timer)
        atomic.AddUint32(&p.stats.Timeouts, 1)
        return ErrPoolTimeout
    }
}
```

3、通过 `popIdle` 方法，尝试从连接池的空闲连接队列 `p.idleConns` 中获取一个已有连接，如果该连接已过期，则关闭并丢弃该连接，继续重复相同尝试操作，直至获取到一个连接或连接队列为空为止 <br>
```golang
// 取出连接
func (p *ConnPool) popIdle() *Conn {
    if len(p.idleConns) == 0 {
        return nil
    }

    idx := len(p.idleConns) - 1
    // 栈的数据结构，取最后一个连接
    cn := p.idleConns[idx]
    p.idleConns = p.idleConns[:idx]
    p.idleConnsLen--
    // 每次都强行补充连接至 MinIdleConns
    p.checkMinIdleConns()
    return cn
}
```

4、如果上一步无法获取到已有连接，则新建一个连接，如果没有返回错误则直接返回，如果新建连接时返回错误，则释放掉轮转队列中的位置，返回连接错误 <br>

完整的 `Get` 方法实现如下：
```golang
// Get returns existed connection from the pool or creates a new one.
func (p *ConnPool) Get(ctx context.Context) (*Conn, error) {
    if p.closed() {
        // 首先，检查连接池是否被关闭，如果被关闭直接返回 `ErrClosed` 错误
        return nil, ErrClosed
    }

    err := p.waitTurn(ctx)
    if err != nil {
        return nil, err
    }

    for {
        p.connsMu.Lock()
        cn := p.popIdle()
        p.connsMu.Unlock()

        if cn == nil {
            break
        }

        // 如果连接已经过期，那么强行关闭此链接，然后重新从空闲队列中获取
        if p.isStaleConn(cn) {
            _ = p.CloseConn(cn)
            continue
        }

        atomic.AddUint32(&p.stats.Hits, 1)
        return cn, nil
    }

    atomic.AddUint32(&p.stats.Misses, 1)
    // 从池中无法获取连接，则新建一个连接，如果没有返回错误则直接返回，如果新建连接时返回错误，则释放掉轮转队列中的位置，返回连接错误
    newcn, err := p.newConn(ctx, true)
    if err != nil {
        p.freeTurn()
        return nil, err
    }

    return newcn, nil
}

func (p *ConnPool) getTurn() {
    p.queue <- struct{}{}
}


func (p *ConnPool) freeTurn() {
    <-p.queue
}
```

####    3、向连接池放回连接
从 `ConnPool` 中取出的连接一般来说都是要放回到连接池中的，具备包含 2 点：
-   直接向空闲连接队列中插入这个连接，并把轮转队列的资源释放掉
-   若连接的标记 `cn.pooled` 为不要被池化，则会直接释放这个连接

```golang
func (p *ConnPool) Put(cn *Conn) {
    // 检查连接中是否还有数据没被读取
    if cn.rd.Buffered()> 0 {
        // 若有，移除连接并返回错误
        internal.Logger.Printf("Conn has unread data")
        p.Remove(cn, BadConnError{})
        return
    }

    // 判断连接是否已经放入连接池中
    if !cn.pooled {
        // 若无，直接移除连接
        p.Remove(cn, nil)
        return
    }

    // 连接队列上锁，将该连接加入空闲连接队列中，连接队列解锁，工作连接通道移除一个元素
    p.connsMu.Lock()
    p.idleConns = append(p.idleConns, cn)
    p.idleConnsLen++
    p.connsMu.Unlock()
    p.freeTurn()
}
```

####    4、移除和关闭连接
删除方法 `removeConn` 会从连接池的 `conns` 队列中移除这个连接，在 `ConnPool` 的实现中，移除（`Remove`）和关闭连接（`CloseConn`），底层调用的都是 `removeConnWithLock` 函数，删除的方法是比较指针的值，然后进行 slice 的删除操作：

```go
func (p *ConnPool) Remove(cn *Conn, reason error) {
    p.removeConnWithLock(cn)
    p.freeTurn()
    _ = p.closeConn(cn)
}

func (p *ConnPool) CloseConn(cn *Conn) error {
    p.removeConnWithLock(cn)
    return p.closeConn(cn)
}
```

removeConnWithLock 函数的工作流程如下：
```go
func (p *ConnPool) removeConnWithLock(cn *Conn) {
    // 连接队列加锁
    p.connsMu.Lock()
    p.removeConn(cn)
    //unlock
    p.connsMu.Unlock()
}

func (p *ConnPool) removeConn(cn *Conn) {
    // 遍历连接队列找到要关闭的连接，并将其移除出连接队列
    for i, c := range p.conns {
        if c == cn {
            // 比较指针的值
            p.conns = append(p.conns[:i], p.conns[i+1:]...)
            if cn.pooled {
                // 如果 cn 这个连接是在 pool 中的，更新连接池统计数据
                p.poolSize--
                // 检查连接池最小空闲连接数量，如果不满足最小值，需要异步补充
                p.checkMinIdleConns()
            }
            return
        }
    }
}

// 关闭连接
func (p *ConnPool) closeConn(cn *Conn) error {
    // 若定义了回调函数（创建连接池时的配置选项传入）
    if p.opt.OnClose != nil {
        _ = p.opt.OnClose(cn)
    }
    // 真正关闭连接
    return cn.Close()
}
```

####    5、监控 && 回收过期连接
连接池是怎么自动收割长时间不使用的空闲连接的？注意到在 `NewConnPool` 方法中，若传入的 `Option` 配置了空闲连接超时和检查空闲连接频率，则新启动一个用于检查并清理过期连接的 goroutine，每隔 `frequency` 时间遍历检查（并取出）连接池中是否存在过期连接，对过期连接做删除和关闭连接操作，并释放轮转资源。其代码如下：

```go
//frequency 指定了多久进行一次检查，这里直接作为定时器 ticker 的触发间隔
func (p *ConnPool) reaper(frequency time.Duration) {
    // 创建 ticker
    ticker := time.NewTicker(frequency)
    defer ticker.Stop()

    for {
        select {
            // 循环判断计时器是否到时
        case <-ticker.C:
            // It is possible that ticker and closedCh arrive together,
            // and select pseudo-randomly pick ticker case, we double
            // check here to prevent being executed after closed.
            if p.closed() {
                return
            }
            // 移除空闲连接队列中的过期连接
            _, err := p.ReapStaleConns()
            if err != nil {
                internal.Logger.Printf("ReapStaleConns failed: %s", err)
                continue
            }
        case <-p.closedCh:
            // 连接池是否关闭
            //pool 的结束标志触发，子协程退出常用范式
            return
        }
    }
}

// 移除空闲连接队列中的过期连接
func (p *ConnPool) ReapStaleConns() (int, error) {
    var n int
    for {
        p.getTurn()

        p.connsMu.Lock()
        cn := p.reapStaleConn()
        p.connsMu.Unlock()
        p.freeTurn()

        if cn != nil {
            // 关闭单个连接
            _ = p.closeConn(cn)
            n++
        } else {
            break
        }
    }
    atomic.AddUint32(&p.stats.StaleConns, uint32(n))
    return n, nil
}
```

#### 6、检查连接是否已经过期
```golang
func (p *ConnPool) isStaleConn(cn *Conn) bool {
	if p.opt.IdleTimeout == 0 && p.opt.MaxConnAge == 0 {
        // 未配置超时选项
		return false
	}

	now := time.Now()
	if p.opt.IdleTimeout > 0 && now.Sub(cn.UsedAt()) >= p.opt.IdleTimeout {
		return true
	}
	if p.opt.MaxConnAge > 0 && now.Sub(cn.createdAt) >= p.opt.MaxConnAge {
		return true
	}

	return false
}
```

####    7、如何剔除失败连接和过期连接
go-redis 在每次执行命令失败以后，会判断当前失败类型，如果不是 redis server 的报错，也不是设置网络设置的timeout 报错，那么则会将该连接从连接池中 remove 掉，如果有设置重试次数，那么就会继续重试命令，又因为每次执行命令时会从连接池中获取连接，而没有又会新建，这样就实现了失败重连和自动剔除机制。
```golang
func (c *baseClient) releaseConn(ctx context.Context, cn *pool.Conn, err error) {
   if c.opt.Limiter != nil {
      c.opt.Limiter.ReportResult(err)
   }
   if isBadConn(err, false, c.opt.Addr) {
       // 连接失败则移除
      c.connPool.Remove(ctx, cn, err)
   } else {
      c.connPool.Put(ctx, cn)
   }
}

// 判断是否属于无效连接
func isBadConn(err error, allowTimeout bool, addr string) bool {
   switch err {
   case nil:
      return false
   case context.Canceled, context.DeadlineExceeded:
      return true
   }

   if isRedisError(err) {
      switch {
      case isReadOnlyError(err):
         // Close connections in read only state in case domain addr is used
         // and domain resolves to a different Redis Server. See #790.
         return true
      case isMovedSameConnAddr(err, addr):
         // Close connections when we are asked to move to the same addr
         // of the connection. Force a DNS resolution when all connections
         // of the pool are recycled
         return true
      default:
         return false
      }
   }

   if allowTimeout {
      if netErr, ok := err.(net.Error); ok &amp;&amp; netErr.Timeout() {
         return !netErr.Temporary()
      }
   }

   return true
}
```

####    8、如何清理过期连接
清理过期连接的实现比较简单，每次从`idleConns`的切片头部取出一个来判断是否过期，如果过期的话，更新`idleConns`，并且关闭过期链接（连接过期时间可以通过参数设置）
```golang
// 清理过期连接
func (p *ConnPool) reapStaleConn() *Conn {
    if len(p.idleConns) == 0 {
        return nil
    }

    // 取出队头的连接
    cn := p.idleConns[0]
    if !p.isStaleConn(cn) {
        //true 已过期
        return nil
    }

    // 从 idleConns 移除队头
    p.idleConns = append(p.idleConns[:0], p.idleConns[1:]...)
    p.idleConnsLen--
    // 从 p.conns 中移除对应的连接
    p.removeConn(cn)

    return cn
}
```


##	0x05 一些细节

####    1、连接池 Keepalive 机制
个人认为，Keepalive 的机制的连接池的设计中是一定需要的，这样一方面可以在应用层对长连接进行保活，另一方面，在服务器出问题（主动断开的情况），客户端也能够及时感知到并可以通知使用方。从连接池里面取出的连接都是可用的连接了。看似简单的代码，却完美的解决了连接池里面超时连接的问题。同时，就算遇到 Redis 服务器重启等情况，也能保证连接自动重连。不过 go-redis 库中并未实现 Keepalive 的功能。

####    2、处理指定的连接
实质上是遍历连接池中的所有连接，并调用传入的 `fn` 过滤函数作用在每个连接上，过滤出符合业务要求的连接。
```go
func (p *ConnPool) Filter(fn func(*Conn) bool) error {
    var firstErr error
    p.connsMu.Lock()
    for _, cn := range p.conns {
        if fn(cn) {
            if err := p.closeConn(cn); err != nil && firstErr == nil {
                firstErr = err
            }
        }
    }
    p.connsMu.Unlock()
    return firstErr
}
```

####    3、统计连接池数据
监控统计对调整连接池配置选项，优化连接池性能提供了重要的依据，在任何系统的设计中都是不必可少的组件。
```go
// 连接池连接数量总数
func (p *ConnPool) Len() int {
    p.connsMu.Lock()
    n := len(p.conns)
    p.connsMu.Unlock()
    return n
}

// 连接池空闲连接数量
func (p *ConnPool) IdleLen() int {
    p.connsMu.Lock()
    n := p.idleConnsLen
    p.connsMu.Unlock()
    return n
}

func (p *ConnPool) Stats() *Stats {
    idleLen := p.IdleLen()
    return &Stats{
        //Hits：连接池命中空闲连接次数
        Hits:     atomic.LoadUint32(&p.stats.Hits),
        //Misses：连接池没有空闲连接可用次数
        Misses:   atomic.LoadUint32(&p.stats.Misses),
        //Timeouts：请求连接等待超时次数
        Timeouts: atomic.LoadUint32(&p.stats.Timeouts),
        //TotalConns：连接池总连接数量
        TotalConns: uint32(p.Len()),
        //IdleConns：连接池空闲连接数量
        IdleConns:  uint32(idleLen),
        //StaleConns：移除过期连接数量
        StaleConns: atomic.LoadUint32(&p.stats.StaleConns),
    }
}
```

####    4、保存 / 获取最后一次错误
通常，连接错误记录是读多写少的，所以采用读写锁来保证该记录的并发安全（读写锁在该场景下性能更佳）。
```go
func (p *ConnPool) getLastDialError() error {
    // 加锁
    p.lastDialErrorMu.RLock()
    err := p.lastDialError
    p.lastDialErrorMu.RUnlock()
    return err
}
```

####    5、使用 `sync.Pool` 复用 timer 对象
```go
var timers = sync.Pool{
	New: func() interface{} {
		t := time.NewTimer(time.Hour)
		t.Stop()
		return t
	},
}
```

##  0x06    上层调用

####    如何初始化一个连接池
go-redis 初始化时采用的 Lazy-Init 的方式，对于当创建一个 Client 后，并不会直接创建连接，而只是初始化相关的 option 属性，并返回一个没有任何连接的 Client
```golang
func NewClient(opt *Options) *Client {
       opt.init()
       c := Client{
              baseClient: baseClient{
                     opt:      opt,
                     connPool: newConnPool(opt),
              },
       }
       c.baseClient.init()
       c.init()

       return c
}
```

####    Options.Dialer方法的意义


##  0x07    思考
分析完上述代码，思考下go-redis的连接池的获取/回收机制会不会存在问题呢？看起来是有的，参见如下:
-   [LIFO connection pool does not give every connection equal chance of being chosen](https://github.com/go-redis/redis/issues/1819)
-   [Allow FIFO pool in redis client](https://github.com/go-redis/redis/pull/1820)

场景是连接池后端连接多个IP时，但是从连接池获取连接的过程中出现了请求负载不均衡的问题。问题的本质在于，当前版本v7的实现是基于LIFO（后进先出）机制，**即每次从空闲连接的切片中都是取出最后一个连接，客户端用完连接后，又会放回到最后一个，这种先入后出的栈的数据结构，导致只要某个连接不过期，永远都是从栈顶取元素，这就会导致不均衡的问题。**

####    优化
目前，在v8[版本](https://github.com/go-redis/redis/releases/tag/v8.11.1)中已经引入了FIFO机制来优化该场景；具体是在options参数中增加了先入先出的取连接方式

```golang
type Options struct {
   Dialer  func(context.Context) (net.Conn, error)
   OnClose func(*Conn) error
   // 增加了先入先出的队列结构
   PoolFIFO           bool
   PoolSize           int
   MinIdleConns       int
   MaxConnAge         time.Duration
   PoolTimeout        time.Duration
   IdleTimeout        time.Duration
   IdleCheckFrequency time.Duration
}
```

获取连接操作优化机制如下代码所示。**从连接池取连接的操作就变成了轮询的方式，较好的解决了连接不均衡的问题**。
```golang
func (p *ConnPool) popIdle() (*Conn, error) {
   if p.closed() {
      return nil, ErrClosed
   }
   n := len(p.idleConns)
   if n == 0 {
      return nil, nil
   }

   var cn *Conn
   if p.opt.PoolFIFO {
      // 若开启了该参数，则从队列头取连接
      cn = p.idleConns[0]
      copy(p.idleConns, p.idleConns[1:])
      p.idleConns = p.idleConns[:n-1]   //注意这段代码，有点意思
   } else {
      // 默认从队列尾部取连接
      idx := n - 1
      cn = p.idleConns[idx]
      p.idleConns = p.idleConns[:idx]
   }
   p.idleConnsLen--
   p.checkMinIdleConns()
   return cn, nil
}
```

##  0x08    总结

##	0x09   参考
-	[Redis Clients](http://redis.io/clients#go)
-   [Go Redis 连接池实现](https://jackckr.github.io/golang/go-redis%E8%BF%9E%E6%8E%A5%E6%B1%A0%E5%AE%9E%E7%8E%B0/)
-	[openresty 连接池](https://wiki.jikexueyuan.com/project/openresty/web/conn_pool.html)
-   [实现连接池的几种姿势](https://zhuanlan.zhihu.com/p/47480504)

转载请注明出处，本文采用 [CC4.0](http://creativecommons.org/licenses/by-nc-nd/4.0/) 协议授权