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

客户端使用 `ab` 进行压测：`ab -n 10 -c 2 http://127.0.0.1:9191/test`，总请求数设置为 `10`，并发量为 `2`，从执行来看，平均每 1s 执行一次 CGI 逻辑。结果如下：
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
2.  为啥 `leakyBucketRateLimiter` 的逻辑中不需要调用 `AbortWithStatus` 返回 `Code 429` 错误
3.  为什么后面的请求耗时越来越久（`>4s`）
4.	如何实现一个健壮的 gin 限流器中间件

接下来，我们从源码分析中探索这几个问题。

##  0x04    Uber 的实现
[ratelimit](https://github.com/uber-go/ratelimit) 是漏桶的一个典型实现。提供了 `atomic` 和 `mutex` 两个版本：
-	atomic 版本：以 `atomic.StorePointer`+`unsafe.Pointer` 来实现原子操作（`https://github.com/uber-go/ratelimit/blob/master/ratelimit.go`）
-	mutex 版本：以 mutex 来控制并发（`https://github.com/uber-go/ratelimit/blob/master/mutexbased.go`）

为了方便理解，这里分析下 `mutex` 的版本：

####	核心结构
```golang
type mutexLimiter struct {
	sync.Mutex
	last       time.Time
	sleepFor   time.Duration
	perRequest time.Duration
	maxSlack   time.Duration
	clock      Clock
}
```

####    初始化
默认 ratelimter 的限速周期是 `time.Second` 即 `1s`，参数中 `rate` 是限速周期内的限速总数，直面上理解为将 `1s` 化为 `rate` 个区间，`maxSlack` 为最大松弛量，下面的篇幅再做解释。
```golang
// New returns a Limiter that will limit to the given RPS.
func newMutexBased(rate int, opts ...Option) Limiter {
	l := &mutexLimiter{
		perRequest: time.Second / time.Duration(rate),
		maxSlack:   -10 * time.Second / time.Duration(rate),	// 最大松弛量
	}
	if l.clock == nil {
		l.clock = clock.New()
	}
	return l
}
```
在初始化代码中：`limiter.perRequest = time.Second / time.Duration(rate)`，这里 `limiter.perRequest` 指的就是每个请求之间的间隔时间。`newMutexBased` 方法中传入的参数 `rate` 就是每秒允许请求量 (RPS) 的值。

####    核心限速逻辑 Take
本小节分析下 Uber Leaky Bucket 的核心限速逻辑，  New(10) 传入的 10 指的是 1s 内只有能有 10 个请求通过, 于是算出来每个请求之间应该间隔 100 ms. 如果两个请求之间间隔时间过短, 那么需要第二个请求 sleep 一段时间, 这样保证请求能够匀速从桶内流出. 如下图如下图，当请求 1 处理结束后, 我们记录下请求 1 的处理完成的时刻, 记为 limiter.last。
稍后请求 2 到来, 如果此刻的时间与 limiter.last 相比并没有达到 perRequest 的间隔大小，那么 sleep 一段时间即可。
![img](https://wx1.sbimg.cn/2020/08/17/3JVNl.png)

由于漏桶的核心限速逻辑是实现每秒固定速率的目的，其实还是比较简单的。
`t.last` 保存了上一次请求通过的最后时间
```golang
// Take blocks to ensure that the time spent between multiple
// Take calls is on average time.Second/rate.
func (t *mutexLimiter) Take() time.Time {
	t.Lock()
	defer t.Unlock()

	now := t.clock.Now()

	// If this is our first request, then we allow it.
	// 如果是第一次请求, 直接放行即可
	if t.last.IsZero() {
		t.last = now
		return t.last
	}

	// sleepFor calculates how much time we should sleep based on
	// the perRequest budget and how long the last request took.
	// Since the request may take longer than the budget, this number
	// can get negative, and is summed across requests.
	// 注意这段代码，累加 sleepFor 的值
	t.sleepFor += t.perRequest - now.Sub(t.last)

	// We shouldn't allow sleepFor to get too negative, since it would mean that
	// a service that slowed down a lot for a short period of time would get
	// a much higher RPS following that.
	if t.sleepFor < t.maxSlack {
		t.sleepFor = t.maxSlack
	}

	// If sleepFor is positive, then we should sleep now.
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

####	maxSlack 应用
首先，最大松弛量的引入是解决什么问题？漏斗桶有个天然缺陷就是无法应对突发流量（匀速，两次请求 `req1` 和 `req2` 之间的延迟至少应该 `>=perRequest`）, 对于突发这种情况，uber-go 对 Leaky Bucket 做了一些改良，引入了最大松弛量（maxSlack）的概念, 表示允许抵消的最长时间：

上面计算 sleepFor 的第，不使用最大松弛量的 sleep 逻辑，严格要求两个请求之间必须间隔 `t.perRequest` 的时间，那么计算 `sleepFor` 的代码就是：

```golang
t.sleepFor = t.perRequest - now.Sub(t.last)
```

这样会有什么问题呢？我们假设现在有三个请求，`req1`、`req2` 和 `req3`，限速区间为 `10ms`。<br>
`req1` 最先到来，当 `req1` 完成之后 `15ms` ，`req2` 才到来，依据限速策略可以对 `req2` 立即处理，当 `req2` 完成后，`5ms` 后， `req3` 到来，这个时候距离上次请求还不足 `10ms`，因此还需要等待 `5ms` 才能继续执行, 但是，对于这种情况，实际上这三个请求一共消耗了 `25ms` 才完成，并不是预期的 `20ms`<br>

上面这个 case 模拟了一种请求量突发的状况。在本算法中，可以把之前间隔比较长的请求的时间（累加超出 `perRequest` 的时延），匀给后面的请求判断限流时使用。 对于上面的 case，因为 `req2` 相当于多等了 `5ms`，我们可以把这 `5ms` 移给 `req3` 使用。加上  `req3` 本身就是 `5ms` 之后过来的，一共刚好 `10ms`，所以 `req3` 无需等待，直接可以处理。此时三个请求也恰好一共是 `20ms`。

此外，注意这段代码：
```golang
// 当计算出来的 sleepFor 超过一个 maxSlack 时，那么就只 sleep 一个 maxSlack 时间，用于两次请求时间间隔过大的场景
if t.sleepFor < t.maxSlack {
	t.sleepFor = t.maxSlack
}
```
但是也有特殊情况, 假设计算出来的间隔时间为 `10ms`, 但是 `req1` 和 `req2` 之间的间隔时间已经远超过 `10ms`（`sleepFor` 远远超出 `10ms`），这段代码就是避免这种情况发生。


##  0x05    总结
本文分析了 Uber 的 Leaky Bucket 漏桶算法的应用及实现。

##  0x06    参考
-   [Leaky_bucket-WIKI](https://en.wikipedia.org/wiki/Leaky_bucket)
-   [流量调整和限流技术](https://colobu.com/2014/11/13/rate-limiting/)
-   [限流器系列 (1) -- Leaky Bucket 漏斗桶](https://www.haohongfan.com/post/2020-06-27-leaky-bucket/)
-   [uber-go 漏桶限流器使用与原理分析](https://www.cyhone.com/articles/analysis-of-uber-go-ratelimit/)

