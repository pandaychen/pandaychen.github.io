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

这里有几个细节需要注意：<br>
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
至此，单实例 Redis 分布式的锁基本成型了，但是，如果 Redis 节点宕机了，那么所有客户端就都无法获得锁了，服务变得不可用。为了提高可用性，我们可以给这个 Redis 节点挂一个 Slave，当 Master 节点不可用的时候，系统自动切到 Slave 上（Failover 机制）。但由于 <font color="#dd0000"> Redis 的主从复制（Replication）是异步的 </font>，这可能导致在 Failover 过程中丧失锁的安全性（此模型存在明显的竞态条件, 会导致多个客户端同时持有一个锁）考虑下面的执行顺序：

1.	客户端 A 从 Master 获取了锁 `lock_key`
2.	在 Master 将锁 `lock_key` 信息备份到 Slave 节点之前, Master 主节点宕机，即存储锁的 `lock_key` 还没有来得及同步到 Slave 上
3.	此时，Slave 节点升级为 Master 节点
4.	客户端 B 从新的 Master 节点获得锁 `lock_key`
5.	客户端 B 从新的 Master 节点获取到了对应同一个资源的锁 `lock_key`
6.	于是，客户端 A 和客户端 B 同时持有了同一个资源的锁 `lock_key`，分布式锁逻辑失效

所以，这就是为什么不能用主从复制实现故障转移的原因，针对这个问题，Redis 的作者 antirez 设计了 [Redlock 算法](https://redis.io/topics/distlock#the-redlock-algorithm)，让用户基于 Redis 集群来实现可靠的分布式锁。

##	0x02 改进：Redlock 算法
RedLock 算法用来解决单实例 Redis 集群的可用性问题，此算法基于 N 个完全独立（非 Redis Cluster）的 Redis Master 节点，客户端来完成获取（分布式）锁的操作。假设 `N == 5`，来看下基于 Redlock 算法 <font color="#dd0000"> 如何获取到过期时间 </font> 为 $t_{final}$ 的分布式锁：

1.	需要申请分布式锁的客户端，获取当前时间（毫秒数），即获取当前机器的毫秒级的时间戳，记为 $t_{cur1}$
2.	分布式锁的过期时间（客户端自行设置）初始值设置为 $t_{expect}$
3.	按顺序依次向 `N` 个 Redis Master 节点 <font color="#dd0000"> 执行获取锁的操作 </font>。这个获取操作跟前面基于单 Redis 节点的获取锁的过程相同，包含随机字符串 `randomValue`（即以相同的 `lock_key` 和随机 `randomValue`），同时也包含过期时间 (比如 `PX 30000`，即锁的有效时间)。为了保证在某个 Redis 节点不可用的时候算法能够继续运行，这个获取锁的操作还有一个超时时间（整体操作超时），它要远小于锁的有效时间（几十毫秒量级，比如锁的有效时间是 `10` 秒，那么这个超时时间大概为 `5~50` 毫秒）。客户端在向某个 Redis 节点获取锁失败以后，应该立即尝试下一个 Redis 节点。这里的失败，应该包含任何类型的失败，比如该 Redis 节点不可用，或者该 Redis 节点上的锁已经被其它客户端持有（注：Redlock 原文中这里只提到了 Redis 节点不可用的情况，但也应该包含其它的失败情况）
4.	计算整个获取锁的过程总共消耗了多长时间，计算方法是用当前时间（记为 $t_{cur2}$）减去第 `1` 步记录的时间（结果记为 $t_{delta}$）。如果客户端从大多数 Redis 节点（即：超过 `Quorum>=`$\frac{N}{2}+1$ 台）成功获取到了锁，并且获取锁总共消耗的时间（即 $t_{delta}$）没有超过锁的有效时间（即 $t_{expect}$），那么这时客户端才认为最终获取锁成功；否则，认为最终获取锁失败。
5.	如果最终获取锁成功了，那么这个锁的有效时间应该重新计算，它等于最初的锁的有效时间减去第 `4` 步计算出来的获取锁消耗的时间，即分布式锁的过期时间（有效时间）为： $t_{final} = t_{expect}-t_{delta}$
6.	如果最终获取锁失败了（可能由于获取到锁的 Redis 节点个数少于 $\frac{N}{2}+1$，或者整个获取锁的过程消耗的时间超过了锁的最初有效时间），那么 <font color="#dd0000"> 客户端应该立即向所有 Redis 节点发起释放锁的操作 </font>（即前面介绍的 Redis Lua 脚本）。释放锁的操作就很简单了, 只需要一步: 客户端向所有 Redis 节点发起释放锁的操作，不管这些节点当时在获取锁的时候成功与否

以上是 Redlock 算法的主要流程，还得考虑到多机的时钟不同，多台机器之间的时钟可能稍微有点偏差，但是不会有太大的偏差的情况，所以在有效性的时长上再扣除偏移量 `CLOCKDRIFT` （可以设置为有效性时长的 `1%` 左右），这样更加安全。那么最终的时间 $t_{final}= t_{expect} – t_{delta} - CLOCKDRIFT$。

对于分布式锁，还有一些容灾的考虑，比如集群 `5` 台机器，突然多启动了一台 Redis 节点，那么整体变成了 `6` 台，如果这台机器可以立刻提供服务，那么有可能两个客户端都能获取到锁（每个客户端都获取了 `3` 个机器的锁），这种情况，有个很好的解决办法是新启动的机器有段时间的冷却，在一个 TTL 之后才能提供服务，也就是大部分锁已经过了有效期之后，再提供服务。

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
redsync 的通用结构定义如下：
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
`Redsync` 结构的成员 `pools` 是个 `redis.Pool` 数组，每个 `redis.Pool` 都是上面的 `Pool` 实现，它代表了一个 Redis 实例的连接池：
```golang
// Redsync provides a simple method for creating distributed mutexes using multiple Redis connection pools.
type Redsync struct {
	pools []redis.Pool	// 对应上面的 Pool，一个 Pool 对应一个 Redis 实例
}
```

####	Mutex 实现
`Mutex` 是分布式锁的定义，代表了一个分布式锁，其成员多为 redlock 算法所需要的条件：
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
Lock 方法实现了 Redlock 算法的加锁接口（Redlock 算法的核心实现），根据上面的算法描述实现，代码逻辑并不难懂。
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
			// 注意：
			// 失败重试： 当客户端无法获取锁得时候会设置一个随机值重试，这个随机值的重试时间应当和当次申请锁的时间错开，减少脑裂的可能。
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

注意上面的 `time.Sleep(m.delayFunc(i))` 的失败重试逻辑，当客户端无法获取锁得时候会设置一个随机值重试，这个随机值的重试时间应当和当次申请锁的时间错开，减少脑裂的可能。此外，一个客户端在所有 Redis 实例中申请的时间越短，发生脑裂的时间窗口越小，所以要用非阻塞的方式，这里 `actOnPoolsAsync` 同时向多个 redis 实例异步发送 Set 请求（实际上是异步发送请求，阻塞获取每个请求的结果），接下来看下 `actOnPoolsAsync` 方法的实现。

####	actOnPoolsAsync 方法
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
外部接口都在 [此](https://github.com/go-redsync/redsync/blob/master/redsync.go)，`NewMutex` 为初始化方法，关注下其中的参数的初始化值：
```golang
// NewMutex returns a new distributed mutex with given name.
func (r *Redsync) NewMutex(name string, options ...Option) *Mutex {
	m := &Mutex{
		name:         name,
		expiry:       8 * time.Second,	// 锁的过期（有效）时间
		tries:        32,				// 最大重试次数
		delayFunc:    func(tries int) time.Duration { return 500 * time.Millisecond },	// 每次失败后，重试 500ms
		genValueFunc: genValue,
		factor:       0.01,				//CLOCKDRIFT 的比率
		quorum:       len(r.pools)/2 + 1,		//quorum 数目
		pools:        r.pools,
	}
	for _, o := range options {
		o.Apply(m)
	}
	return m
}
```

##	0x06	Redsync 的使用
Redsync 的使用基本上和前述一致，注意每个 `Pool` 的连接池中连接的数目：
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
本文简单介绍单机 Redis 分布式锁的缺点以及多机 Redis 分布式锁算法 Redlock 算法的实现。在项目如何选择 [合适的分布式锁算法呢](https://books.studygolang.com/advanced-go-programming-book/)？
1.	业务还在单机就可以搞定的量级时，那么按照需求使用任意的单机锁方案就可以
2.	如果发展到了分布式服务阶段，但业务规模不大，比如 `qps < 1000`，使用哪种锁方案都差不多。如果公司内已有可以使用的 Zookeeper/Etcd/Redis 集群，那么就尽量在不引入新的技术栈的情况下满足业务需求
3.	业务发展到一定量级的话，就需要从多方面来考虑了。首先是你的锁是否在任何恶劣的条件下都不允许数据丢失，如果不允许，那么就不要使用 Redis 的 setnx 的简单锁；如果要使用 Redlock 算法，那么要考虑 Redis 的集群方案的安全问题，即是否可以直接把对应的 Redis 的实例的 `ip/port` 暴露给开发人员。如果不可以，那也没法用
4.	对锁数据的可靠性要求极高的话，那只能使用 Etcd 或者 Zookeeper 这种通过一致性协议保证数据可靠性的锁方案。但可靠的背面往往都是较低的吞吐量和较高的延迟。需要根据业务的量级对其进行压力测试，以确保分布式锁所使用的 Etcd/Zookeeper 集群可以承受得住实际的业务请求压力。需要注意的是，Etcd 和 Zookeeper 集群是没有办法通过增加节点来提高其性能的。要对其进行横向扩展，只能增加搭建多个集群来支持更多的请求。这会进一步提高对运维和监控的要求。此外，多个集群可能需要引入 Proxy，没有 Proxy 那就需要业务去根据某个业务 id 来做 sharding。如果业务已经上线的情况下做扩展，还要考虑数据的动态迁移。这些都不是容易的事情。在选择具体的方案时，还是需要多加思考，对风险早做预估。

##  0x08	参考
-   [Distributed locks with Redis](https://redis.io/topics/distlock)
-   [Distributed mutual exclusion lock using Redis for Go](https://github.com/go-redsync/redsync)
-   [How to do distributed locking](http://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html)
-	[Go 语言高级编程：6.1 分布式锁](https://books.studygolang.com/advanced-go-programming-book/ch6-cloud/ch6-01-lock.html)
