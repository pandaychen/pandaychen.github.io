---
layout:     post
title:      gRPC 应用篇之客户端 Connection Pool（未完成）
subtitle:   连接池的实现分析 && 我们是否需要 Tcp/gRPC 客户端连接池？
date:       2020-10-03
author:     pandaychen
header-img: img/golang-horse-fly.png
catalog: true
tags:
    - gRPC
---

##  0x00    连接池
之前分析过 `go-redis` 的连接池实现：[Go-Redis 连接池（Pool）源码分析](https://pandaychen.github.io/2020/02/22/A-REDIS-POOL-ANALYSIS/)，在项目应用中，连接池的最大好处是减少 TCP 建立握手 / 挥手的时间，实现 TCP 连接复用，从而降低通信时延和提高性能。

通常一些高性能中间件都提供了内置的 TCP 连接池，如刚才说的 `go-redis`,[`go-sql-driver`](https://github.com/go-sql-driver/mysql#connection-pool-and-timeouts) 等等，关于连接池，一个良好的设计是对用户屏蔽底层的实现，如存储 / keepalive / 关闭 / 自动重连等，只对用户提供简单的获取接口。


##  0x01  gRPC 连接池的实现


##	0x02	gRPC Pool 分析
滴滴开源的 [grpc 连接池](https://github.com/shimingyah/pool)，代码不长。简单分析下：

####    grpc.conn 封装
代码主要 [在此](https://github.com/shimingyah/pool/blob/master/conn.go)，
```golang
// Conn single grpc connection inerface
type Conn interface {
	// Value return the actual grpc connection type *grpc.ClientConn.
	Value() *grpc.ClientConn

	// Close decrease the reference of grpc connection, instead of close it.
	// if the pool is full, just close it.
	Close() error
}

// Conn is wrapped grpc.ClientConn. to provide close and value method.
type conn struct {
	cc   *grpc.ClientConn       // 封装真正的 grpc.conn
	pool *pool                  // 指向的 pool
	once bool
}
```

####    Pool 封装
gRPC-Pool 封装的主要代码 [在此](https://github.com/shimingyah/pool/blob/master/pool.go)，一个 `interface`，一个 `struct`：<br>
Pool 对外部暴露的接口就 `3` 个：
-	`Get`：从连接池获取连接
-	`Close`：关闭连接池
-	`Status`：打印连接池信息

```golang
// Pool interface describes a pool implementation.
// An ideal pool is threadsafe and easy to use.
type Pool interface {
	// Get returns a new connection from the pool. Closing the connections puts
	// it back to the Pool. Closing it when the pool is destroyed or full will
	// be counted as an error. we guarantee the conn.Value() isn't nil when conn isn't nil.
	Get() (Conn, error)     // 从池中取连接

	// Close closes the pool and all its connections. After Close() the pool is
	// no longer usable. You can't make concurrent calls Close and Get method.
	// It will be cause panic.
	Close() error           // 关闭池，关闭池中的连接

	// Status returns the current status of the pool.
	Status() string
}

// gRPC 连接池的定义
type pool struct {
	// atomic, used to get connection random
	index uint32
	// atomic, the current physical connection of pool
	current int32
	// atomic, the using logic connection of pool
	// logic connection = physical connection * MaxConcurrentStreams
	ref int32
	// pool options
	opt Options
    // all of created physical connections
    //  真正存储连接的结构
	conns []*conn
	// the server address is to create connection.
	address string
	// control the atomic var current's concurrent read write.
	sync.RWMutex
}
```

```golang
// New return a connection pool.
func New(address string, option Options) (Pool, error) {
	if address == "" {
		return nil, errors.New("invalid address settings")
	}
	if option.Dial == nil {
		return nil, errors.New("invalid dial settings")
	}
	if option.MaxIdle <= 0 || option.MaxActive <= 0 || option.MaxIdle> option.MaxActive {
		return nil, errors.New("invalid maximum settings")
	}
	if option.MaxConcurrentStreams <= 0 {
		return nil, errors.New("invalid maximun settings")
	}

	p := &pool{
		index:   0,
		current: int32(option.MaxIdle),
		ref:     0,
		opt:     option,
		conns:   make([]*conn, option.MaxActive),
		address: address,
	}

	for i := 0; i < p.opt.MaxIdle; i++ {
		c, err := p.opt.Dial(address)
		if err != nil {
			p.Close()
			return nil, fmt.Errorf("dial is not able to fill the pool: %s", err)
		}
		p.conns[i] = p.wrapConn(c, false)
	}
	log.Printf("new pool success: %v\n", p.Status())

	return p, nil
}

func (p *pool) incrRef() int32 {
	newRef := atomic.AddInt32(&p.ref, 1)
	if newRef == math.MaxInt32 {
		panic(fmt.Sprintf("overflow ref: %d", newRef))
	}
	return newRef
}

func (p *pool) decrRef() {
	newRef := atomic.AddInt32(&p.ref, -1)
	if newRef < 0 {
		panic(fmt.Sprintf("negative ref: %d", newRef))
	}
	if newRef == 0 && atomic.LoadInt32(&p.current) > int32(p.opt.MaxIdle) {
		p.Lock()
		if atomic.LoadInt32(&p.ref) == 0 {
			log.Printf("shrink pool: %d ---> %d, decrement: %d, maxActive: %d\n",
				p.current, p.opt.MaxIdle, p.current-int32(p.opt.MaxIdle), p.opt.MaxActive)
			atomic.StoreInt32(&p.current, int32(p.opt.MaxIdle))
			p.deleteFrom(p.opt.MaxIdle)
		}
		p.Unlock()
	}
}

func (p *pool) reset(index int) {
	conn := p.conns[index]
	if conn == nil {
		return
	}
	conn.reset()
	p.conns[index] = nil
}

func (p *pool) deleteFrom(begin int) {
	for i := begin; i < p.opt.MaxActive; i++ {
		p.reset(i)
	}
}

// Get see Pool interface.
func (p *pool) Get() (Conn, error) {
	// the first selected from the created connections
	nextRef := p.incrRef()
	p.RLock()
	current := atomic.LoadInt32(&p.current)
	p.RUnlock()
	if current == 0 {
		return nil, ErrClosed
	}
	if nextRef <= current*int32(p.opt.MaxConcurrentStreams) {
		next := atomic.AddUint32(&p.index, 1) % uint32(current)
		return p.conns[next], nil
	}

	// the number connection of pool is reach to max active
	if current == int32(p.opt.MaxActive) {
		// the second if reuse is true, select from pool's connections
		if p.opt.Reuse {
			next := atomic.AddUint32(&p.index, 1) % uint32(current)
			return p.conns[next], nil
		}
		// the third create one-time connection
		c, err := p.opt.Dial(p.address)
		return p.wrapConn(c, true), err
	}

	// the fourth create new connections given back to pool
	p.Lock()
	current = atomic.LoadInt32(&p.current)
	if current <int32(p.opt.MaxActive) && nextRef > current*int32(p.opt.MaxConcurrentStreams) {
		// 2 times the incremental or the remain incremental
		increment := current
		if current+increment > int32(p.opt.MaxActive) {
			increment = int32(p.opt.MaxActive) - current
		}
		var i int32
		var err error
		for i = 0; i < increment; i++ {
			c, er := p.opt.Dial(p.address)
			if er != nil {
				err = er
				break
			}
			p.reset(int(current + i))
			p.conns[current+i] = p.wrapConn(c, false)
		}
		current += i
		log.Printf("grow pool: %d ---> %d, increment: %d, maxActive: %d\n",
			p.current, current, increment, p.opt.MaxActive)
		atomic.StoreInt32(&p.current, current)
		if err != nil {
			p.Unlock()
			return nil, err
		}
	}
	p.Unlock()
	next := atomic.AddUint32(&p.index, 1) % uint32(current)
	return p.conns[next], nil
}

// Close see Pool interface.
func (p *pool) Close() error {
	atomic.StoreUint32(&p.index, 0)
	atomic.StoreInt32(&p.current, 0)
	atomic.StoreInt32(&p.ref, 0)
	p.deleteFrom(0)
	log.Printf("close pool success: %v\n", p.Status())
	return nil
}

// Status see Pool interface.
func (p *pool) Status() string {
	return fmt.Sprintf("address:%s, index:%d, current:%d, ref:%d. option:%v",
		p.address, p.index, p.current, p.ref, p.opt)
}
```


####    grpc.Pool 的使用
本小节给出基于 gRPC 连接池的 CS 调用例子，如下：<br>

服务端代码：
```golang
func main() {
	flag.Parse()

	listen, err := net.Listen("tcp", fmt.Sprintf("127.0.0.1:%v", *port))
	if err != nil {
		log.Fatalf("failed to listen: %v", err)
	}

	// 调整 grpc 的默认参数
	s := grpc.NewServer(
		grpc.InitialWindowSize(pool.InitialWindowSize),
		grpc.InitialConnWindowSize(pool.InitialConnWindowSize),
		grpc.MaxSendMsgSize(pool.MaxSendMsgSize),
		grpc.MaxRecvMsgSize(pool.MaxRecvMsgSize),
		grpc.KeepaliveEnforcementPolicy(keepalive.EnforcementPolicy{
			PermitWithoutStream: true,
		}),
		grpc.KeepaliveParams(keepalive.ServerParameters{
			Time:    pool.KeepAliveTime,
			Timeout: pool.KeepAliveTimeout,
		}),
	)
	pb.RegisterEchoServer(s, &server{})

	if err := s.Serve(listen); err != nil {
		log.Fatalf("failed to serve: %v", err)
	}
}
```

客户端代码：
```golang
func main() {
	flag.Parse()

	p, err := pool.New(*addr, pool.DefaultOptions)
	if err != nil {
		log.Fatalf("failed to new pool: %v", err)
	}
	defer p.Close()

	conn, err := p.Get()
	if err != nil {
		log.Fatalf("failed to get conn: %v", err)
	}
	defer conn.Close()

	client := pb.NewEchoClient(conn.Value())
	ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
	defer cancel()

	res, err := client.Say(ctx, &pb.EchoRequest{Message: []byte("hi")})
	if err != nil {
		log.Fatalf("unexpected error from Say: %v", err)
	}
	fmt.Println("rpc response:", res)
}
```

##  0x03	通用 TCP 连接池的实现
基于前面的分析，如何实现一个通用的 Tcp 连接池呢？



##  0x05	参考
-   [一个典型的 tcp 连接池开源实现](https://github.com/silenceper/pool)
-   [gRPC 连接池](https://github.com/shimingyah/pool)
-   [gRPC 连接池的设计与实现](https://zhuanlan.zhihu.com/p/100200985)
-   [grpc-go-pool](https://github.com/processout/grpc-go-pool)
-	[Golang 连接池的几种实现案例](https://zhuanlan.zhihu.com/p/109852936)