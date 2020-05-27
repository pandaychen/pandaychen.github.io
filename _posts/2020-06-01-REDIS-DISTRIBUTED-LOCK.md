---
layout:     post
title:      使用 Redis 实现分布式锁
subtitle:   Redis 应用开发
date:       2020-06-01
author:     pandaychen
catalog:    true
tags:
    - 分布式锁
---


##  0x00    前言
前文中，分析了如何使用Etcd来构建分布式锁，这篇文章，来聊聊如何使用Redis构造分布式锁。一个高可用的分布式锁至少需要满足 ** 三个条件 **：
1. 互斥：在任意时间内，只有一个客户端能够获得一把锁；
2. 避免死锁。即使客户端宕机或者从集群中分离了，其他客户端依然有可能获取到锁；
3. 容错。只要大部分的 Redis 节点存活，客户端就能正确的获取锁和释放锁；

对于 redis 而言，上述三个条件都非常容易满足，非常适合做分布式锁服务。

##  0x01 单机 Redis 锁


####    加锁
对于单机 Redis 而言，获取锁的方式非常简单，使用SET（SET if not exists）指令完成（当然lock_name必须唯一），该条指令只允许被一个Redis-Client设置成功：
```bash
SET lock_name random_value NX PX 30000
```

##	0x02	单机 Redis 锁不安全


##	0x03 改进：Redlock 算法
实际的 Redlock 算法（ Redis集群），是在 N 个 master 上实现的，假设 N 为 5，来看看如何获取到过期时间为 t0 的分布式锁。

1.	请求锁的客户端，获取当前机器的毫秒级的时间戳，计为 $$t_1$$；锁的过期时间初始值设置为$$t_0$$
2.	在所有的机器（客户端）上以**相同的 key** 和**随机 value** 值获取锁，这里可以并行也可以串行，但是必须得设置一个获取锁的超时时间，避免获取锁的时间太长。这个超时时间相对锁的有效时间需要相对较短，比如锁的有效时间是 10 秒，那么这个超时时间大概为 5~50 毫秒

3.	如何判断是否获取分布式锁成功？具体计算方式是判断客户端获取成功了几台机器的锁，如果大于至少一半的机器（N=5 时，至少为 3 台），并且获取锁的有效时间大于 0，那么就获取锁成功。有效时间的计算方法为：计当前时间戳为$$t_2$$，用 $$t_2$$ 的减去时间戳 $$t_1$$ 获得时间消耗为 $$dt$$，有效性的时间要大于 $$dt$$ 才行，那么锁的有效时长 $$VT= t_0 – t_1$$

假设获取分布式锁失败了，必须要释放掉获取成功的节点上的锁。如果有重试机制，需要在一段随机值之后，再进行重试获取锁。
以上则是 redlock 的主要算法流程，还得考虑到多机的时钟不同，多台机器之间的时钟可能稍微有点偏差，但是不会有太大的偏差，所以在有效性的时长上再扣除一点点偏移量 CLOCK_DRIFT (可以设置为有效性时长的 1% 左右)，这样更加安全，那么 VT= t0 – t1-CLOCK_DRIFT。

对于分布式锁，还有一些容灾的考虑，比如集群 5 台机器，突然又起了一台机器，那么整体变成了 6 台，如果这台机器可以立刻提供服务，那么有可能两个客户端都能获取到锁（每个客户端都获取了 3 个机器的锁），这种情况，有个很好的解决办法是新启动的机器有段时间的冷却，在一个 TTL 之后才能提供服务，也就是大部分锁已经过了有效期之后，再提供服务。


##  0x04    锁的使用
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

##  0x05  Redlock 开源实现
官方推荐的实现是[redsync](https://github.com/go-redsync/redsync)，

Lock方法实现了Redlock算法的加锁接口：

```golang
// Lock locks m. In case it returns an error on failure, you may retry to acquire the lock by calling this method again.
func (m *Mutex) Lock() error {
	value, err := m.genValueFunc()
	if err != nil {
		return err
	}

	for i := 0; i < m.tries; i++ {
		if i != 0 {
			time.Sleep(m.delayFunc(i))
		}

		start := time.Now()

		n, err := m.actOnPoolsAsync(func(pool Pool) (bool, error) {
			return m.acquire(pool, value)
		})
		if n == 0 && err != nil {
			return err
		}

		now := time.Now()
		until := now.Add(m.expiry - now.Sub(start) - time.Duration(int64(float64(m.expiry)*m.factor)))
		if n >= m.quorum && now.Before(until) {
			m.value = value
			m.until = until
			// 申请Redlock锁成功
			return nil
		}
		m.actOnPoolsAsync(func(pool Pool) (bool, error) {
			return m.release(pool, value)
		})
	}

	return ErrFailed
}   
```

注意，在`actOnPoolsAsync`方法中传入封装的`REDIS-SET`指令：
```golang
func (m *Mutex) acquire(pool Pool, value string) (bool, error) {
	conn := pool.Get()
	defer conn.Close()
	reply, err := redis.String(conn.Do("SET", m.name, value, "NX", "PX", int(m.expiry/time.Millisecond)))
	if err != nil {
		if err == redis.ErrNil {
			return false, nil
		}
		return false, err
	}
	return reply == "OK", nil
}
```


```golang
func (m *Mutex) actOnPoolsAsync(actFn func(Pool) (bool, error)) (int, error) {
	type result struct {
		Status bool
		Err    error
	}

	ch := make(chan result)
	for _, pool := range m.pools {
		go func(pool Pool) {
			r := result{}
			r.Status, r.Err = actFn(pool)
			ch <- r
		}(pool)
	}
	n := 0
	var err error
	for range m.pools {
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

##  0x06	总结
本文简单介绍单机 redis 分布式锁的缺点以及多机 redis 分布式锁的实现方式。

##  0x07	参考
-   [Distributed locks with Redis](https://redis.io/topics/distlock)
-   [Distributed mutual exclusion lock using Redis for Go](https://github.com/go-redsync/redsync)
-   [How to do distributed locking](http://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html)