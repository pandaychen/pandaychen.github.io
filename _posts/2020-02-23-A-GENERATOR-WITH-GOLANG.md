---
layout:     post
title:      使用 Golang 开发 generator（生成器）
subtitle:   生成器：一种程序逻辑优化的思路
date:       2020-02-23
author:     pandaychen
header-img: img/golang-horse-fly.png
catalog: true
category:   false
tags:
    - Golang
    - OpenSSH
---

##  0x00    简介
&emsp;&emsp; Python 中提供了 `yield` 关键字，用以实现生成器（generator）的功能。如下，计算 fibonacci 数的生成器：
```python
def fib(max):
    n,a,b =0,0,1
    while n < max:
        yield b
        a,b =b,a+b
        n = n+1
    return 'done'

a = fib(10)
print(fib(10))
```

在笔者的项目中，遇到这样的场景，用来生成 `RSA-2048bit` OpenSSH 秘钥的方法，在 4 核（core），CPU 为 `Intel(R) Xeon(R) CPU E5-2680 v4 @ 2.40GHz`，平均需要 200-300ms 左右，这对于每次 SSH 登录都需要调用的方法来说，耗时还是比较久的（不必要的耗时）。如何来优化这种场景呢？

##  0x01    解决的方法
&emsp;&emsp; 对于解决这种需要临时产生并拿来用的数据的应用场景，一个（预先 Prepared）生产者 + 消费者的模型就很适合这种场景的优化。程序启动时，生产者就开始生产 OpenSSH 秘钥，放入缓冲区 Buffer 中，待消费者来取，这样每次的耗时可以大大降低了（当然，这里不排除有消费者速度超过生产者速度的情况，需要增加缓冲区或者增多生产者来优化）。

我们先设计下 generator 需要实现的特性：
1.  多 Producer + 多 Consumer
2.  线程安全
3.  无阻塞，缓冲区无数据可取时，返回错误（降级，让消费者自己去生产）

配合 Golang 的 Channel 机制的线程安全特性，可以很方便的实现上述特性。

##  0x02    OpenSSH 秘钥生成器
基于 Channel 实现的 generator 的核心代码如下：
```golang
type SshKey struct {
	Publickey string
	Seckey    string
}

type SshKeyGenerator struct {
	//start, step int
	Size    int
	Queue   chan SshKey
	Running bool
}

func NewGenerator(size int) *SshKeyGenerator {
	gtr := &SshKeyGenerator{
		Size:    size,
		Running: true,
		Queue:   make(chan SshKey, size),
	}
	//start producer
	go gtr.process()
	return gtr
}

func (g *SshKeyGenerator) process() {
	defer func() { recover() }()
	var i = 0
	for {
		// 先判断
		sk, pk, err := g.NewRsaKeyPair()
		if err != nil {
			fmt.Println("Generator key pairs error", err.Error())
			time.Sleep(1 * time.Second)
			continue
		}
		skeyitem := SshKey{
			Publickey: pk,
			Seckey:    sk,
		}
		select {
		case g.Queue <- skeyitem:
			i++
			//if full,block
		}
	}
}

func (g *SshKeyGenerator) Next() *SshKey {
	select {
	case it := <-g.Queue:
		return &it
	default:
		return nil
	}
}

func (g *SshKeyGenerator) Close() {
	g.Running = false
	close(g.Queue)
}
```

##	0x03    其他的应用场景

####	SnowFlake 算法
snowflake 算法，是 twitter 开源的唯一 ID 生成算法是生成器最佳的应用场景。主要解决了高并发时 ID 生成不重复的问题，它满足了 Twitter 每秒上万条消息的请求，使每条消息有唯一、有一定顺序的 ID ，且支持分布式生成。


##	0x04    参考
-   [python 生成器和迭代器有这篇就够了](https://www.cnblogs.com/wj-1314/p/8490822.html)
-	[Twitter snowflake ID 算法之 golang 实现](https://segmentfault.com/a/1190000013831352)

转载请注明出处，本文采用 [CC4.0](http://creativecommons.org/licenses/by-nc-nd/4.0/) 协议授权