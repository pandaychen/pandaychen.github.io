---
layout:     post
title:      Go-redis：高级用法（未完）
subtitle:   go-redis With SLA
date:       2020-08-20
header-img: img/super-mario.jpg
author:     pandaychen
catalog:    true
tags:
    - Redis
---

##  0x00    前言
本文总结下，工作中使用 [go-redis](https://github.com/go-redis/redis) 库的一些高级用法。


##  0x01    回调钩子 WrapProcess
V7 版本之前提供了 `WrapProcess` 方法，用于在执行方法前后进行自定义处理，下面的例子中，使用 `time.Since()` 来计算 Redis 操作的耗时代码：
注意：`err := old(cmd)` 这段代码指传入的真正的 Redis 执行方法。

```golang
func main() {
	cl := redis.NewClient(&redis.Options{
			Addr: ":6379",
	})
	cl.WrapProcess(func(old func(cmd redis.Cmder) error) func(cmd redis.Cmder) error {
			return func(cmd redis.Cmder) error {
					fmt.Printf("starting process:<%s>\n", cmd)
					start := time.Now()
					//old 这里代指 redis 操作
					err := old(cmd)
					fmt.Printf("finished process:<%s>,cost:%v\n", cmd,time.Since(start).Seconds())

					return err
			}
	})

	//get
	Get := func(client *redis.Client, key string) *redis.StringCmd {
			cmd := redis.NewStringCmd("get", key)
			client.Process(cmd)
			return cmd
	}

	//set
	Set := func(client *redis.Client, key string, value string) *redis.StringCmd {
			cmd := redis.NewStringCmd("set", key, value)
			client.Process(cmd)
			return cmd
	}

	//use
	_, errSet := Set(cl, "myKey", "myValue").Result()
	if errSet != nil {
			fmt.Println("redis set failed.", errSet)
	}
	value, errGet := Get(cl, "myKey").Result()
	if errGet != nil {
			fmt.Println("redis get failed.", errGet)
	} else {
			fmt.Println("The key value is", value)
	}

	//something else
	cmd := redis.NewStringCmd("set", "myKey1", "123", "ex", "100")
	cl.Process(cmd)

	//get expire
	cmd1 := redis.NewIntCmd("ttl", "myKey1")
	cl.Process(cmd1)
	expire, errEx := cmd1.Result()
	if errEx != nil {
			fmt.Println("ttl failed.", errEx)
	} else {
			fmt.Println("expire of key is", expire)
	}
}
```

上面这段代码输出为：
```bash
10 0
starting process:<set myKey myValue:>
finished process:<set myKey myValue: OK>,cost:0.011436692
starting process:<get myKey:>
finished process:<get myKey: myValue>,cost:6.601e-05
The key value is myValue
starting process:<set myKey1 123 ex 100:>
finished process:<set myKey1 123 ex 100: OK>,cost:5.202e-05
starting process:<ttl myKey1: 0>
finished process:<ttl myKey1: 100>,cost:3.9867e-05
expire of key is 100
```


不过，在 [V7 版本](https://github.com/go-redis/redis/blob/736fa2865992feedee4f744ff3e6e5d2c27a17f3/CHANGELOG.md) 中，该方法已经被替换为 [`AddHook`](https://github.com/go-redis/redis/blob/master/redis.go#L44)：
>   WrapProcess is replaced with more convenient AddHook that has access to context.Context.

```golang
func (hs *hooks) AddHook(hook Hook) {
	hs.hooks = append(hs.hooks, hook)
}
```


##	0x02	Redis metrics
如上一小节所述，怎么将项目中的 Redis 操作的细粒度指标（如操作延迟、连接池或者 pipeline 或者一般操作的失败次数等）接入到 Prometheus 中呢？
答案就是 `WrapProcess`+`Prometheus Exportor`，可以编写单独的 exportor，封装 `WrapProcess` 的方法，实现对 redis 操作的监控。

##  0x03	连接池的用法



##	参考
-	[聊聊 GO-REDIS 的一些高级用法](http://vearne.cc/archives/1113)
-	[Monitoring Redis Clusters with Prometheus](https://www.metricfire.com/blog/monitoring-redis-clusters-with-prometheus/)