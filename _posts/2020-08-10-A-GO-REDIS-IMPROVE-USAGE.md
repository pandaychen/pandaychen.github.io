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
V7 版本之前提供了 `WrapProcess` 和 `WrapProcessPipeline` [方法](https://github.com/go-redis/redis/pull/1011/files)，用于在 RedisAPI 执行方法前后进行自定义处理，下面的例子中，使用 `time.Since()` 来计算 Redis 操作的耗时代码：
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

我们按照前文 [Prometheus 应用接入：使用 Prometheus 开发 Exporter](https://pandaychen.github.io/2020/07/13/ANOTHER-LOOK-AT-PROMETHEUS/) 中定义的 `4` 步来实现这个 `Exporter`：

1.	实现定好需要采集哪些指标，如成功率，延迟，分布数据等
2.	将指标转换为 Prometheus 的 Metrics 类型（或者直接使用 Prometheus 的内置类型）并完成注册
3.	在代码逻辑中加入 Metrics 的 "打点" 调用
4.	启动 `PrometheusHttp` 服务，暴露自己的采集的指标

####	定义 Redis Metrics 结构
由于我们要统计 Redis 操作耗时，那么操作延迟可能会分布在不同的区间上，使用 `HistogramVec` 结构来进行记录是比较合适的。定义 `RedisCollector` 结构如下（这里我们使用一个 `interface{}` 来存储 `redis.Client`）：
```golang
type RedisClient interface {
	WrapProcess(fn func(oldProcess func(cmd redis.Cmder) error) func(cmd redis.Cmder) error)
	WrapProcessPipeline(func(old func([]redis.Cmder) error) func([]redis.Cmder) error)
}

type RedisCollector struct {
	Client    *RedisClient
	once sync.Once
	execDurationHistogram  *prometheus.HistogramVec	// 用于收集耗时数据
}
```

####	注册 metrics
```golang
var _redisCollector *RedisCollector

func init() {
	_redisCollector = &RedisCollector{
		Client: new(RedisClient),
	}

	_redisCollector.execDurationHistogram = prometheus.NewHistogramVec(prometheus.HistogramOpts{
		Name:    "redis_exec_duration_seconds",
		Help:    "Redis command exec duration in seconds",
		Buckets: []float64{1e-03, 2.5e-03, 5e-03, 10e-03, 25e-03, 50e-03, 100e-03, 250e-03, 500e-03, 1, 2.5, 5, 10},
	}, []string{"role"})
	prometheus.MustRegister(_redisCollector.execDurationHistogram)
}
```

####	Metrics 打点 + 上报
这里我们实现的 `AddRedisExecDuration` 方法，调用 `WrapProcess` 和 `WrapProcessPipeline`，加入计时的逻辑，如下代码所示：
```golang
func AddRedisExecDuration(client RedisClient) {
	client.WrapProcess(func(old func(cmd redis.Cmder) error) func(cmd redis.Cmder) error {
		return func(cmd redis.Cmder) error {
			start := time.Now()
			defer func() {
				_redisCollector.execDurationHistogram.WithLabelValues(role).Observe(time.Since(start).Seconds())
			}()

			return old(cmd)
		}
	})

	client.WrapProcessPipeline(func(old func([]redis.Cmder) error) func([]redis.Cmder) error {
		return func(cmds []redis.Cmder) error {
			start := time.Now()
			defer func() {
				_redisCollector.execDurationHistogram.WithLabelValues(role).Observe(time.Since(start).Seconds())
			}()

			return old(cmds)
		}
	})
}
```


####	启动 http 服务和暴露指标
使用 `promhttp.Handler()` 注册 HTTP 服务，开启 `50` 个 goroutine 模拟业务逻辑：
```golang
func main() {
    client := redis.NewClient(&redis.Options{
        Addr:     "localhost:6379",
        PoolSize: 50,
    })

    metric.AddRedisExecDuration(client)

	// 模拟业务负载
    for i := 0; i < 50; i++ {
        go func() {
            for {
                client.Get("a").String()
                time.Sleep(200 * time.Millisecond)
            }
        }()
    }

    http.Handle("/metrics", promhttp.Handler())
    log.Fatal(http.ListenAndServe(":8080", nil))
}
```

####	查询指标
为了模拟耗时，在 `AddRedisExecDuration` 加入 `sleep` 随机操作来模拟，通过 `curl http://127.0.0.1:8080/metrics` 获取某个时间点的指标如下：
```bash
# HELP redis_request_duration_seconds Redis request duration in seconds
# TYPE redis_request_duration_seconds histogram
redis_request_duration_seconds_bucket{role="test",le="0.001"} 0
redis_request_duration_seconds_bucket{role="test",le="0.0025"} 0
redis_request_duration_seconds_bucket{role="test",le="0.005"} 0
redis_request_duration_seconds_bucket{role="test",le="0.01"} 0
redis_request_duration_seconds_bucket{role="test",le="0.025"} 3
redis_request_duration_seconds_bucket{role="test",le="0.05"} 6
redis_request_duration_seconds_bucket{role="test",le="0.1"} 10
redis_request_duration_seconds_bucket{role="test",le="0.25"} 26
redis_request_duration_seconds_bucket{role="test",le="0.5"} 57
redis_request_duration_seconds_bucket{role="test",le="1"} 120
redis_request_duration_seconds_bucket{role="test",le="2.5"} 206
redis_request_duration_seconds_bucket{role="test",le="5"} 206
redis_request_duration_seconds_bucket{role="test",le="10"} 206
redis_request_duration_seconds_bucket{role="test",le="+Inf"} 206
redis_request_duration_seconds_sum{role="test"} 186.20372700699994
redis_request_duration_seconds_count{role="test"} 206
```


##	0x03	参考
-	[Prometheus Exporter for Redis Metrics. Supports Redis 2.x, 3.x, 4.x, 5.x and 6.x](https://github.com/oliver006/redis_exporter)
-	[Monitoring Redis Clusters with Prometheus](https://www.metricfire.com/blog/monitoring-redis-clusters-with-prometheus/)