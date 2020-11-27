---
layout:     post
title:      分布式锁：使用 Redis 实现
subtitle:   Redis 应用开发：分布式锁构建 && Redlock 实现分析
date:       2020-06-01
author:     pandaychen
img:	img/super-mario.jpg
catalog:    true
tags:
	- 分布式锁
	- Redis
---

##  0x00    前言
在应用环境中，不同的进程需要以互斥的方式使用共享资源时，就必须用到分布式锁。前文中，分析了如何 [使用 Etcd 来构建分布式锁](https://pandaychen.github.io/2019/10/24/ETCD-DISTRIBUTED-LOCK/)。本文，来聊聊如何使用 Redis 构造（可靠 && 高可用）分布式锁。一个高可用的分布式锁 <font color="#dd0000"> 至少需要满足几个条件 </font>：
1. 互斥：在任意时间内，只有一个客户端能够获得一把锁，具有排它性
2. 避免死锁：即使客户端宕机或者从集群中分离了，其他客户端依然有可能获取到锁
3. 容错规则：只要大部分的节点（Redis）存活，客户端就能正确的获取锁和释放锁
4. 容错规则：客户端最终一定可以获得和释放锁，即使锁住某个资源的客户端在释放锁之前崩溃或者发生网络分区

对于 Redis （高可用集群）而言，上述三个条件都非常容易满足，非常适合做分布式锁服务。

##  0x01 单实例 Redis 锁演进
单实例 Redis 一般通过 `SET` 指令设置锁，该指令表示只有当 `lock_key` 不存在的时候才能 `SET` 成功，`lock_key` 代表了共享资源的唯一锁标识：
```bash
SET lock_key(resource_name) random_value NX PX TIMEOUT
```

这里有几个细节需要注意：
1、`random_value` 必须为随机（每个客户端生成自己唯一的 `random_value`） <br>
为什么 `random_value` 不随机的方式会有问题？考虑如下场景：
1.	客户端 A 获取锁 `lock_key` 成功
2.	客户端 A 在某个操作上阻塞了很长时间（超过 `TIMEOUT`），过期时间到了，锁自动释放了
3.	客户端 B 获取到了对应同一个资源的锁（超时 `lock_key/random_value` 被删除）
4.	客户端 A 从阻塞中恢复过来，释放掉了客户端 B 持有的锁
5.	后续流程中，客户端 B 在访问共享资源的时候，就没有锁为它提供保护了，分布式锁逻辑失效

2、释放锁必须使用 lua 原子操作 <br>
在获取锁时 `SET lock_key(resource_name) random_value NX PX TIMEOUT` 是原子的, 在释放锁的时候同样需要保证其操作的原子性，如果不采用原子操作，会有下面的问题：
1.	客户端 A 获取锁 `lock_key` 成功
2.	客户端 A 访问共享资源
3.	客户端 A 为了释放锁，先执行 `GET` 操作获取 `lock_key` 的值
4.	客户端 A 判断随机字符串的值，与预期的值相等
5.	客户端 A 由于某个原因阻塞住了很长时间
6.	过期时间到了，`lock_key/resource_name` 被删除，锁也自动释放了
7.	客户端 B 获取到了对应同一个资源的锁
8.	客户端 A 从阻塞中恢复过来，继续执行 DEL 操作，释放掉了客户端 B 持有的锁，分布式锁逻辑失效

注意：阻塞的场景包括业务 BUG 或者出现较大的网络延迟，这在现网中都是较大概率的事件。

3、超时的时间？<br>
既然 lock_key 是存储在 Redis，那么必须加上 TTL, 假设业务处理时间为 `N` 秒, 那么我们的 TTL 设置为 `N+1 ~ 2*N` 即可，这样即可以避免锁提前释放, 也可以防止释放锁失败的情况下, 分布式锁长时间不可用

4、单机 Redis 的弊端 <br>
至此，单实例 Redis 分布式的锁基本成型了，但是，如果 Redis 节点宕机了，那么所有客户端就都无法获得锁了，服务变得不可用。为了提高可用性，我们可以给这个 Redis 节点挂一个 Slave，当 Master 节点不可用的时候，系统自动切到 Slave 上（Failover）。但由于 <font color="#dd0000"> Redis 的主从复制（Replication）是异步的 </font>，这可能导致在 Failover 过程中丧失锁的安全性（此模型存在明显的竞态条件, 会导致多个客户端同时持有一个锁）考虑下面的执行顺序：

1.	客户端 A 从 Master 获取了锁 `lock_key`
2.	在 Master 将锁 `lock_key` 信息备份到 Slave 节点之前, Master 主节点宕机，即存储锁的 `lock_key` 还没有来得及同步到 Slave 上
3.	此时，Slave 节点升级为 Master 节点
4.	客户端 B 从新的 Master 节点获得锁 `lock_key`
5.	客户端 B 从新的 Master 节点获取到了对应同一个资源的锁 `lock_key`
6.	于是，客户端 A 和客户端 B 同时持有了同一个资源的锁 `lock_key`，分布式锁逻辑失效

所以，这就是为什么不能用主从复制实现故障转移的原因，针对这个问题，redis 的作者 antirez 设计了 [Redlock 算法]()，让用户基于 Redis 集群来实现可靠的分布式锁。

##	0x02 改进：Redlock 算法
RedLock 算法用来解决单实例 Redis 集群的可用性问题，此算法基于 N 个完全独立（非 Redis Cluster）的 Redis 节点，客户端来完成获取锁的操作，算法的基础描述如下：


##  0x03    锁的应用
一般客户端使用分布式锁的方式为：
```golang
//1.    定义服务器的地址列表
servers := []string{"serveraddr"}
//2.    创建（初始化）锁
dlock := NewDistributedLock(servers)
//3.    申请锁
ret := dlock.Lock("lockname", lock_time)
//4.    执行业务逻辑
DoBusiness()
//5.    释放锁
dlock.Unlock(ret)
```

##  0x04  Redlock 开源实现：redsync
[官方](https://redis.io/topics/distlock) 推荐的实现是 [redsync](https://github.com/go-redsync/redsync)，本小节分析下其实现。redsync 支持下面两个 redis 客户端库：
* [Redigo](https://github.com/gomodule/redigo)
* [Go-redis](https://github.com/go-redis/redis)

兼容的方式就是采用 `interface{}` 抽象出 [Pool 的接口](https://github.com/go-redsync/redsync/blob/master/redis/redis.go)，具体方法的实现细节由每个 [不同的库实现](https://github.com/go-redsync/redsync/blob/master/redis/goredis/goredis.go)。

####	通用定义
如下：
-	`Pool`：抽象连接池
-	`Conn`：抽象每个 Redis 连接
-	`Script`：Redis 脚本
```golang
// A Pool maintains a pool of Redis connections.
type Pool interface {
	Get(ctx context.Context) (Conn, error)
}

type Conn interface {
	Get(name string) (string, error)
	Set(name string, value string) (bool, error)
	SetNX(name string, value string, expiry time.Duration) (bool, error)
	Eval(script *Script, keysAndArgs ...interface{}) (interface{}, error)
	PTTL(name string) (time.Duration, error)
	Close() error
}

type Script struct {
	KeyCount int
	Src      string
	Hash     string
}
```

####	Pool 定义
```golang
// Redsync provides a simple method for creating distributed mutexes using multiple Redis connection pools.
type Redsync struct {
	pools []redis.Pool	// 对应上面的 Pool，一个 Pool 对应一个 Redis 实例
}
```

####	Mutex 实现
Mutex 是分布式锁的定义，
```GOLANG
// A Mutex is a distributed mutual exclusion lock.
type Mutex struct {
	name   string  				// 名称
	expiry time.Duration    	// 锁的有效时间
	tries     int   // 尝试次数
	delayFunc DelayFunc // 失败尝试设置延迟

	factor float64 // 误差系数控制

	quorum int // 投票数 一般为节点数 / 2+1，节点数为奇数

	genValueFunc func() (string, error)  // 加密函数，生成唯一随机串
	value        string  // 默认就是唯一随机串
	until        time.Time // 过期时间

	pools []Pool // 连接池（每个 Pool 指一个 Redis 实例）
}
```

####	获取锁
Lock 方法实现了 Redlock 算法的加锁接口，根据上面的算法描述实现，代码逻辑并不难懂。
```golang
// Lock locks m. In case it returns an error on failure, you may retry to acquire the lock by calling this method again.
func (m *Mutex) LockContext(ctx context.Context) error {
	// 生成 uniq 随机串，默认 base64
	value, err := m.genValueFunc()
	if err != nil {
		return err
	}

	// tries 为尝试次数
	for i := 0; i < m.tries; i++ {
		if i != 0 {
			time.Sleep(m.delayFunc(i))
		}

		start := time.Now()

		// 尝试异步去获取锁（）
		n, err := m.actOnPoolsAsync(func(pool redis.Pool) (bool, error) {
			return m.acquire(ctx, pool, value)
		})
		if n == 0 && err != nil {
			return err
		}

		now := time.Now()
		// 过期时间 = 有效时间值 - 获取锁消耗的时间值 - 有效时间值 * 误差系数
		until := now.Add(m.expiry - now.Sub(start) - time.Duration(int64(float64(m.expiry)*m.factor)))
		// 成功节点数 >= 节点数 / 2+1 && 未过期时，判定加锁成功
		if n >= m.quorum && now.Before(until) {
			// 申请 Redlock 锁成功
			m.value = value
			m.until = until
			return nil
		}

		// 获取锁失败，异步释放锁
		_, _ = m.actOnPoolsAsync(func(pool redis.Pool) (bool, error) {
			return m.release(ctx, pool, value)
		})
	}

	return ErrFailed
}
```

注意上面的 `for i := 0; i < m.tries; i++` 重试逻辑，

`actOnPoolsAsync` 方法封装了向 `m.pools`（保存了所有的 Redis 实例的连接池）发送命令并获取结果的方法：
```golang
func (m *Mutex) actOnPoolsAsync(actFn func(redis.Pool) (bool, error)) (int, error) {
	type result struct {
		Status bool
		Err    error
	}

	ch := make(chan result)
	for _, pool := range m.pools {
		go func(pool redis.Pool) {
			r := result{}
			r.Status, r.Err = actFn(pool)
			ch <- r
		}(pool)
	}
	n := 0
	var err error
	for range m.pools {
		// 阻塞等待返回
		r := <-ch
		if r.Status {
			n++
		} else if r.Err != nil {
			err = multierror.Append(err, r.Err)
		}
	}
	return n, err
}
```

`actOnPoolsAsync` 方法中的参数 `actFn` 有 [两类](https://github.com/go-redsync/redsync/blob/master/mutex.go#L54)：
1、调用 `m.acquire` 去批量设置锁 <br>
```golang
func(pool redis.Pool) (bool, error) {
	return m.acquire(ctx, pool, value)
}

//acquire 加锁
func (m *Mutex) acquire(ctx context.Context, pool redis.Pool, value string) (bool, error) {
	conn, err := pool.Get(ctx)
	if err != nil {
		return false, err
	}
	defer conn.Close()
	// 调用 Redlock 封装的 SetNX 方法加锁（优化 + ctx？）
	reply, err := conn.SetNX(m.name, value, m.expiry)
	if err != nil {
		return false, err
	}
	return reply, nil
}
```

2、调用 `m.release` 批量释放锁 <br>
```golang
func(pool redis.Pool) (bool, error) {
	return m.release(ctx, pool, value)
}

// release 释放锁
func (m *Mutex) release(ctx context.Context, pool redis.Pool, value string) (bool, error) {
	conn, err := pool.Get(ctx)
	if err != nil {
		return false, err
	}
	defer conn.Close()

	// 调用 Eval 以脚本方式释放锁
	status, err := conn.Eval(deleteScript, m.name, value)
	if err != nil {
		return false, err
	}
	return status != 0, nil
}
```

####	释放锁 UnlockContext
和加锁的方法类似，只要释放锁的数量 `n>=m.quorum`，那么可以认为解锁成功：
```golang
// Unlock unlocks m and returns the status of unlock.
func (m *Mutex) UnlockContext(ctx context.Context) (bool, error) {
	n, err := m.actOnPoolsAsync(func(pool redis.Pool) (bool, error) {
		return m.release(ctx, pool, m.value)
	})
	if n < m.quorum {
		// 释放锁失败
		return false, err
	}
	return true, nil
}
```

##	0x05 Mutex 初始化
外部接口都在 [此](https://github.com/go-redsync/redsync/blob/master/redsync.go)
```golang
// NewMutex returns a new distributed mutex with given name.
func (r *Redsync) NewMutex(name string, options ...Option) *Mutex {
	m := &Mutex{
		name:         name,
		expiry:       8 * time.Second,
		tries:        32,
		delayFunc:    func(tries int) time.Duration { return 500 * time.Millisecond },
		genValueFunc: genValue,
		factor:       0.01,
		quorum:       len(r.pools)/2 + 1,
		pools:        r.pools,
	}
	for _, o := range options {
		o.Apply(m)
	}
	return m
}
```

##	0x06	Redsync 的使用

```golang
func main() {
	// Create a pool with go-redis (or redigo) which is the pool redisync will
	// use while communicating with Redis. This can also be any pool that
	// implements the `redis.Pool` interface.
	client := goredislib.NewClient(&goredislib.Options{
		Addr: "localhost:6379",
	})
	pool := goredis.NewPool(client) // or, pool := redigo.NewPool(...)

	// Create an instance of redisync to be used to obtain a mutual exclusion
	// lock.
	rs := redsync.New(pool)

	// Obtain a new mutex by using the same name for all instances wanting the
	// same lock.
	mutexname := "my-global-mutex"
	mutex := rs.NewMutex(mutexname)

	// Obtain a lock for our given mutex. After this is successful, no one else
	// can obtain the same lock (the same mutex name) until we unlock it.
	if err := mutex.Lock(); err != nil {
		panic(err)
	}

	// Do your work that requires the lock.

	// Release the lock so other processes or threads can obtain a lock.
	if ok, err := mutex.Unlock(); !ok || err != nil {
		panic("unlock failed")
	}
}
```


##  0x07	总结
本文简单介绍单机 redis 分布式锁的缺点以及多机 redis 分布式锁的实现方式。

##  0x08	参考
-   [Distributed locks with Redis](https://redis.io/topics/distlock)
-   [Distributed mutual exclusion lock using Redis for Go](https://github.com/go-redsync/redsync)
-   [How to do distributed locking](http://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html)
-	[6.1 分布式锁](https://books.studygolang.com/advanced-go-programming-book/ch6-cloud/ch6-01-lock.html)