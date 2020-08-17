---
layout:     post
title:      开源限流组件分析（一）：Uber 的 Leaky Bucket
subtitle:   分析 Uber 的基于 Leaky Bucket 的限流器
date:       2020-07-31
header-img: img/super-mario.jpg
author:     pandaychen
catalog:    true
tags:
    - 限流
---

##  0x00    前言
前一篇文章了解了微服务中限流的基础概念，这篇文章来分析下基于漏桶（Leaky Bucket）的一个典型开源实现 [Uber：ratelimit](https://github.com/uber-go/ratelimit) 及其一般应用场景。

##  0x01    概念回顾
漏桶算法是对计数器算法的一种改进。
漏桶可以看作是一个带有常量服务时间的单服务器队列，如果漏桶（包缓存）溢出，那么数据包会被丢弃。


##  0x02    Uber-ratelimiter 的使用

####    官方例子
官方提供的例子如下, 设置限流器每秒可以通过 100 个请求，也就是平均每个请求间隔 10ms。漏洞算法的理解起来，相较于令牌桶，可能不那么直观。因为令牌桶是限制 "令牌" 的速率，拿到令牌的请求才能被放行，而漏桶是限制
"单位时间（如 `1s`）" 允许放过的请求数目。<br>
在这里例子中，限制 `1s` 内最多通过的请求数目为 `100`：
```golang
func main() {
    rl := ratelimit.New(100) // per second

    prev := time.Now()
    for i := 0; i < 10; i++ {
        now := rl.Take()
        fmt.Println(i, now.Sub(prev))
        prev = now
    }
}
```

####    CGI 限速
将 ratelimiter 和 gin-CGI 结合，如下面的例子，设置 `1s` 内最多只能放行一个请求（不管实际的 CGI 业务逻辑是快还是慢），如下面的模型所示：
![image](https://wx2.sbimg.cn/2020/08/17/3JLHd.png)
```golang
// 定义全局限流器对象 rl
var rl ratelimit.Limiter

// 初始化 rl
func init() {
    rl = ratelimit.New(1)
}

// 在 gin.HandlerFunc 加入限流逻辑
func leakyBucketRateLimiter() gin.HandlerFunc {
    prev := time.Now()
    return func(c *gin.Context) {
            now := rl.Take()
            fmt.Println(now.Sub(prev))
            //prev = now
    }
}

func main() {
    engine := gin.Default()
    engine.GET("/test", leakyBucketRateLimiter(), func(context *gin.Context) {
            context.JSON(200, true)
    })
    engine.Run("127.0.0.1:8080")
}
```

客户端使用 `ab` 进行压测：`ab -n 10 -c 2 http://127.0.0.1:9191/test`，总请求数设置为 10，并发量为 2，从执行来看，平均每 1s 执行一次 CGI 逻辑。结果如下：
```golang
Benchmarking 127.0.0.1 (be patient)...42.585505171s
[GIN] 2020/08/03 - 06:29:55 | 200 |      38.706µs |       127.0.0.1 | GET      "/test"
1s
[GIN] 2020/08/03 - 06:29:56 | 200 |  999.786807ms |       127.0.0.1 | GET      "/test"
1s
[GIN] 2020/08/03 - 06:29:57 | 200 |  1.999787475s |       127.0.0.1 | GET      "/test"
1s
[GIN] 2020/08/03 - 06:29:58 | 200 |  2.999764691s |       127.0.0.1 | GET      "/test"
1s
[GIN] 2020/08/03 - 06:29:59 | 200 |   3.99970603s |       127.0.0.1 | GET      "/test"
1s
[GIN] 2020/08/03 - 06:30:00 | 200 |  4.999704749s |       127.0.0.1 | GET      "/test"
1s
[GIN] 2020/08/03 - 06:30:01 | 200 |  4.999741308s |       127.0.0.1 | GET      "/test"
1s
[GIN] 2020/08/03 - 06:30:02 | 200 |  4.999620977s |       127.0.0.1 | GET      "/test"
1s
[GIN] 2020/08/03 - 06:30:03 | 200 |  4.999643646s |       127.0.0.1 | GET      "/test"
1s
[GIN] 2020/08/03 - 06:30:04 | 200 |  4.999673156s |       127.0.0.1 | GET      "/test"
..done


Server Software:
Server Hostname:        127.0.0.1
Server Port:            9191

Document Path:          /test
Document Length:        4 bytes

Concurrency Level:      5
Time taken for tests:   9.001 seconds
Complete requests:      10
Failed requests:        0
Write errors:           0
Total transferred:      1260 bytes
HTML transferred:       40 bytes
Requests per second:    1.11 [#/sec] (mean)
Time per request:       4500.286 [ms] (mean)
Time per request:       900.057 [ms] (mean, across all concurrent requests)
Transfer rate:          0.14 [Kbytes/sec] received
```

##  0x03    对漏桶算法的疑问
从上面限流器和 gin 结合的测试代码来看，有几个需要注意的点：
1.  从代码层面上，在 `gin.HandlerFunc` 逻辑 `leakyBucketRateLimiter` 前面都加入 `rl.Take()` 限速控制，这是一个全局控制方法，对应于全局变量 `var rl ratelimit.Limiter`，而 CGI 是一个并发协程模型，这种是非常常用的 golang 设计模式
2.  为啥 `leakyBucketRateLimiter` 的逻辑中不需要返回 `Code 429` 错误
3.  为什么后面的请求耗时越来越久（`>4s`）
4.	如何实现一个健壮的 gin 限流器中间件

接下来，我们从源码分析中探索这几个问题。

##  0x04    Uber 的实现
[ratelimit](https://github.com/uber-go/ratelimit) 是漏桶的一个典型实现。

####    初始化
默认 ratelimter 的限速周期是 `time.Second` 即 `1s`，参数中 `rate` 是限速周期内的限速总数，直面上理解为将 `1s` 化为 `rate` 个区间，`maxSlack` 为最大松弛量，下面再做解释：
```golang
func New(rate int, opts ...Option) Limiter {
	l := &limiter{
		perRequest: time.Second / time.Duration(rate),
		maxSlack:   -10 * time.Second / time.Duration(rate),
	}
	for _, opt := range opts {
		opt(l)
	}
	if l.clock == nil {
		l.clock = clock.New()
    }

	initialState := state{
		last:     time.Time{},
		sleepFor: 0,
    }

    // 使用 atomic 存储
	atomic.StorePointer(&l.state, unsafe.Pointer(&initialState))
	return l
}
```

####    限速核心逻辑
由于漏桶的核心限速逻辑是实现每秒固定速率的目的，其实还是比较简单的。

在 ratelimit 的 New 函数中，传入的参数是每秒允许请求量 (RPS)。
我们可以很轻易的换算出每个请求之间的间隔：

1
limiter.perRequest = time.Second / time.Duration(rate)
以上 limiter.perRequest 指的就是每个请求之间的间隔时间。

如下图，当请求 1 处理结束后, 我们记录下请求 1 的处理完成的时刻, 记为 limiter.last。
稍后请求 2 到来, 如果此刻的时间与 limiter.last 相比并没有达到 perRequest 的间隔大小，那么 sleep 一段时间即可。
```golang
func (t *limiter) Take() time.Time {
	newState := state{}
	taken := false
	for !taken {
		now := t.clock.Now()

		previousStatePointer := atomic.LoadPointer(&t.state)
		oldState := (*state)(previousStatePointer)

		newState = state{}
		newState.last = now

		// If this is our first request, then we allow it.
		if oldState.last.IsZero() {
			taken = atomic.CompareAndSwapPointer(&t.state, previousStatePointer, unsafe.Pointer(&newState))
			continue
		}

		// sleepFor calculates how much time we should sleep based on
		// the perRequest budget and how long the last request took.
		// Since the request may take longer than the budget, this number
		// can get negative, and is summed across requests.
		newState.sleepFor += t.perRequest - now.Sub(oldState.last)
		// We shouldn't allow sleepFor to get too negative, since it would mean that
		// a service that slowed down a lot for a short period of time would get
		// a much higher RPS following that.
		if newState.sleepFor < t.maxSlack {
			newState.sleepFor = t.maxSlack
		}
		if newState.sleepFor > 0 {
			newState.last = newState.last.Add(newState.sleepFor)
		}
		taken = atomic.CompareAndSwapPointer(&t.state, previousStatePointer, unsafe.Pointer(&newState))
	}
	t.clock.Sleep(newState.sleepFor)
	return newState.last
}
```


####    Take 方法
```golang
func (t *limiter) Take() time.Time {
	t.Lock()
	defer t.Unlock()

	now := t.clock.Now()

	// 如果是第一次请求, 直接让通过
	if t.last.IsZero() {
		t.last = now
		return t.last
	}

	// 这里有个最大松弛量的概念 maxSlack
	t.sleepFor += t.perRequest - now.Sub(t.last)
	if t.sleepFor < t.maxSlack {
		t.sleepFor = t.maxSlack
	}

	// 判断是否桶溢出. 如果桶溢出了, 需要 sleep 一段时间
	if t.sleepFor > 0 {
		t.clock.Sleep(t.sleepFor)
		t.last = now.Add(t.sleepFor)
		t.sleepFor = 0
	} else {
		t.last = now
	}
	return t.last
}
```

##  0x05    总结

##  0x06    参考
-   [Leaky_bucket-WIKI](https://en.wikipedia.org/wiki/Leaky_bucket)
-   [流量调整和限流技术](https://colobu.com/2014/11/13/rate-limiting/)
-   [限流器系列 (1) -- Leaky Bucket 漏斗桶](https://www.haohongfan.com/post/2020-06-27-leaky-bucket/)
-   [uber-go 漏桶限流器使用与原理分析](https://www.cyhone.com/articles/analysis-of-uber-go-ratelimit/)

