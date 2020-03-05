---
layout:     post
title:      使用 Golang 开发“生成器”
subtitle:   一种程序逻辑优化的思路
date:       2020-02-23
author:     pandaychen
header-img:
catalog: true
category:   false
tags:
    - Golang
    - 生成器
---


##  0x00    简介
&emsp;&emsp; Python中提供了yield关键字，用以实现生成器（generator）的功能。如计算fibonacci数的生成器：
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

在笔者的项目，也遇到过一个场景，用来生成RSA-2048bit的秘钥`+` X.509 的秘钥的方法，在4核（core），CPU为`Intel(R) Xeon(R) CPU E5-2680 v4 @ 2.40GHz`，平均需要200-300ms左右，这对于每次SSH都需要调用的方法来说，耗时还是比较久的。如何来优化这种场景呢？

##  0x01    思考解决的方法
&emsp;&emsp; 对于这种需要临时产生并拿来用的数据的应用场景，以计算机解决的思路，一个（预先Prepared）生产者 `+` 消费者的模型就很适合这种场景的优化。程序启动时，生产者就开始生产秘钥，放入缓冲区buffer中，待消费者来取，这样每次的耗时可以大大降低了（当然，这里不排除有消费者速度超过生产者速度的情况，需要增加缓冲区或者增多生产者来优化）。

我们先设计下需要实现的特性：
1.  多producer+多consumer
2.  线程安全
3.  无阻塞，缓冲区无数据可取时，返回错误（降级，让消费者自己去生产）

配合 golang-Channel 的线程安全特性，可以很方便的实现上述几点。本文就来实现这个功能。


##	其他的应用场景

####	SnowFlake算法
snowflake算法，是 twitter 开源的唯一 ID 生成算法是生成器最佳的应用场景。主要解决了高并发时 ID 生成不重复的问题，它满足了 Twitter 每秒上万条消息的请求，使每条消息有唯一、有一定顺序的 ID ，且支持分布式生成。


##	参考
-   [python 生成器和迭代器有这篇就够了](https://www.cnblogs.com/wj-1314/p/8490822.html)
-	[Twitter snowflake ID 算法之 golang 实现](https://segmentfault.com/a/1190000013831352)