---
layout:     post
title:      Go-Redis 连接池（Pool）源码分析
subtitle:   分析一款典型的redis连接池实现
date:       2020-02-22
author:     pandaychen
header-img:
catalog: true
category:   false
tags:
    - Redis
---

##  0x00    介绍
&emsp;&emsp;连接池技术，一般是客户端侧高效管理和复用连接，避免重复创建（带来的性能损耗，特别是 `TLS`）和销毁连接的一种技术手段。在项目中灵活使用连接池，对降低服务器负载十分有帮助；此外，在司内的DB托管场景，如遇后台升配、代理扩容等场景，如果服务内置了连接池，一般不需要重启（因为连接池会自动尝试重建）。如 go-xorm 的 [连接池](https://github.com/go-xorm/manual-zh-CN/blob/master/chapter-01/1.engine.md)、go-redis 的 [连接池](https://github.com/go-redis/redis/tree/master/internal/pool)，本文就来分析下 go-redis 中的连接池实现。

总览下连接池的核心 [代码结构](https://github.com/go-redis/redis/blob/master/internal/pool/pool.go)，go-redis 的连接池实现分为如下几个部分：
1.	连接池初始化、管理连接
2.	建立与关闭连接
3.	获取与放回连接
4.	监控统计 && 连接保活配置

##  0x01    Pool 相关的结构体
连接池 [选项](https://github.com/go-redis/redis/blob/master/internal/pool/pool.go#L51) 定义：
```go
type Options struct {
	Dialer  func(context.Context) (net.Conn, error)
	OnClose func(*Conn) error

	PoolSize           int          // 连接池大小
	MinIdleConns       int
	MaxConnAge         time.Duration
	PoolTimeout        time.Duration
	IdleTimeout        time.Duration    // 空闲连接的超时时间
	IdleCheckFrequency time.Duration    // 检查空闲连接频率
}
```

[单个连接](https://github.com/go-redis/redis/blob/master/internal/pool/conn.go) 的结构体定义，核心是 `net.Conn` 封装及根据 Redis 协议实现相应的读写操作：
```go
type Conn struct {
    netConn net.Conn  // tcp 连接

    rd *proto.Reader // 根据 Redis 通信协议实现的 Reader
    wr *proto.Writer // 根据 Redis 通信协议实现的 Writer

    Inited    bool // 是否完成初始化
    pooled    bool // 是否放进连接池
    createdAt time.Time // 创建时间
    usedAt    int64 // 使用时间
}
```

连接池的结构体定义：
```go
type ConnPool struct {
    opt *Options // 连接池配置

    dialErrorsNum uint32 // 连接错误次数，atomic

    lastDialErrorMu sync.RWMutex // 上一次连接错误锁，读写锁
    lastDialError   error // 上一次连接错误

    queue chan struct{}

    connsMu      sync.Mutex // 连接队列锁
    conns        []*Conn // 连接队列
    idleConns    []*Conn // 空闲连接队列
    poolSize     int // 连接池大小
    idleConnsLen int // 空闲连接队列长度

    stats Stats // 连接池统计的结构体

    _closed  uint32 // 连接池关闭标志，atomic
    closedCh chan struct{} // 通知连接池关闭通道（用于主协程通知子协程的常用方法）
}
```

连接池统计 Stats 的结构定义：
```go
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
首先，封装 Pool 的接口实现方法，如下：
```go
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

##  0x03    连接池管理功能

####    1、初始化连接池
使用工厂函数创建全局 Pool 对象，首先会初始化一个 ConnPool 实例，赋予 PoolSize 大小的连接队列和轮转队列，接着会根据 MinIdleConns 参数维持一个最小连接数，以保证连接池中有这么多数量的连接处于活跃状态。IdleTimeout 和 IdleCheckFrequency 参数用来在每过一段时间内会对连接池中不活跃的连接做清理操作。再看 ConnPool 结构体的定义，conns 和 idleConns 这两个队列，conns 就是连接队列，而 idleConns 就是轮转队列。
```go
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
	p.conns = append(p.conns, cn)   //conns 增加
	p.idleConns = append(p.idleConns, cn) //idleConns 增加
	p.connsMu.Unlock()
	return nil
}

```

####    2、关闭连接池
```go
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

##  0x04    单个连接管理

####    1、建立连接
```go
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
```go
// Get returns existed connection from the pool or creates a new one.
func (p *ConnPool) Get(ctx context.Context) (*Conn, error) {
    if p.closed() {
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

        if p.isStaleConn(cn) {
            _ = p.CloseConn(cn)
            continue
        }

        atomic.AddUint32(&p.stats.Hits, 1)
        return cn, nil
    }

    atomic.AddUint32(&p.stats.Misses, 1)

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

func (p *ConnPool) freeTurn() {
    <-p.queue
}

func (p *ConnPool) popIdle() *Conn {
    if len(p.idleConns) == 0 {
        return nil
    }

    idx := len(p.idleConns) - 1
    cn := p.idleConns[idx]
    p.idleConns = p.idleConns[:idx]
    p.idleConnsLen--
    p.checkMinIdleConns()
    return cn
}
```

####    3、向连接池放回连接
```go
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
在 Pool 的实现中，移除（Remove）和关闭连接（CloseConn），底层调用的都是 removeConnWithLock 函数：
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
在 `NewConnPool` 方法中，若传入的 Option 配置了空闲连接超时和检查空闲连接频率，则新启动一个用于检查并清理过期连接的 goroutine，每隔 frequency 时间遍历检查连接池中是否存在过期连接，并清理。其代码如下：
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

//
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

#### 6、检查连接是否已经过期
```go
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

##	0x05 一些细节

####    1、连接池 Keepalive 机制
个人认为，Keepalive 的机制的连接池的设计中是一定需要的，这样一方面可以在应用层对长连接进行保活，另一方面，在服务器出问题（主动断开的情况），客户端也能够及时感知到并可以通知使用方。从连接池里面取出的连接都是可用的连接了。看似简单的代码，却完美的解决了连接池里面超时连接的问题。同时，就算遇到 Redis 服务器重启等情况，也能保证连接自动重连。不过 go-redis 库中并未实现 Keepalive 的功能。

####    2、处理指定的连接
实质上是遍历连接池中的所有连接，并调用传入的 fn 过滤函数作用在每个连接上，过滤出符合业务要求的连接。
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

##	0x06    参考
-	[Redis Clients](http://redis.io/clients#go)
-   [Go Redis 连接池实现](https://jackckr.github.io/golang/go-redis%E8%BF%9E%E6%8E%A5%E6%B1%A0%E5%AE%9E%E7%8E%B0/)
-	[openresty 连接池](https://wiki.jikexueyuan.com/project/openresty/web/conn_pool.html)
-   [实现连接池的几种姿势](https://zhuanlan.zhihu.com/p/47480504)

转载请注明出处，本文采用 [CC4.0](http://creativecommons.org/licenses/by-nc-nd/4.0/) 协议授权