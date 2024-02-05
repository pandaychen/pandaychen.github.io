---
layout: post
title: 分布式限流：基于 Redis 实现
subtitle: 基于 Redis 实现的分布式限流方案总结
date: 2020-09-21
author: pandaychen
catalog: true
tags:
  - Redis
  - 限流
  - 分布式
---

## 0x00 前言

本文梳理下基于 Redis 实现的分布式限流的策略和算法。这里主要关注 <font color="#dd0000"> 分布式 </font> 的场景，主要的实现算法有如下几类：

- 计数器限流
- 固定时间窗口限流
- 滑动时间窗口限流
- 漏桶限流
- 令牌桶限流

## 0x01 Why Redis && Lua？
在一个分布式系统中，存在多个微服务提供服务。所以当瞬间的流量同时访问同一个资源，如何让计数器在分布式系统中正常计数？ 同时在计算资源访问时，可能会涉及多个计算，如何保证计算的原子性？分布式限流本质上是一个集群并发问题，Redis + Lua 的方案非常适合此场景：

1. Redis 单线程特性，适合解决分布式集群的并发问题
2. Redis 本身支持 Lua 脚本执行，可以实现原子执行的效果

Redis 执行 Lua 脚本会以原子性方式进行，在执行脚本时不会再执行其他脚本或命令。并且，Redis 只要开始执行 Lua 脚本，就会一直执行完该脚本再进行其他操作，所以 Lua 脚本中 <font color="#dd0000"> 不能进行耗时操作 </font>。此外，基于 Redis + Lua 的应用场景非常多，如分布式锁，限流，秒杀等等。

基于项目经验来看，使用 Redis + Lua 方案有如下注意事项：

1.  使用 Lua 脚本实现原子性操作的 CAS，避免不同客户端先读 Redis 数据，经过计算后再写数据造成的并发问题
2.  前后多次请求的结果有依赖关系时，最好使用 Lua 脚本将多个请求整合为一个；但请求前后无依赖时，使用 pipeline 方式，比 Lua 脚本方便
3.  为了保证安全性，在 Lua 脚本中不要定义自己的全局变量，以免污染 Redis 内嵌的 Lua 环境。因为 Lua 脚本中你会使用一些预制的全局变量，比如说 `redis.call()`
4.  注意 Lua 脚本的时间复杂度，Redis 的单线程同样会阻塞在 Lua 脚本的执行中，Lua 脚本不要进行高耗时操作
5.  Redis 要求单个 Lua 脚本操作的 key 必须在同一个 Redis 节点上，因此 Redis Cluster 方式需要设置 `HashTag`（实际中不太建议这样操作）


## 0x02 常用方法

本小节介绍下常用的分布式限流方案。

#### 计数器

计数器限流的核心是 `INCRBY` 和 `EXPIRE` 指令，测试用例 [在此](https://github.com/pandaychen/golang_in_action/blob/master/redis/go-redis/scripts_limit1.go)，通常，计数器算法容易出现不平滑的情况，瞬间的 qps 有可能超过系统的承载。

```lua
-- 获取调用脚本时传入的第一个 key 值（用作限流的 key）
local key = KEYS[1]
-- 获取调用脚本时传入的第一个参数值（限流大小）
local limit = tonumber(ARGV[1])
-- 获取计数器的限速区间 TTL
local ttl = tonumber(ARGV[2])

-- 获取当前流量大小
local curentLimit = tonumber(redis.call('get', key) or "0")

-- 是否超出限流
if curentLimit + 1 > limit then
    -- 返回 (拒绝)
    return 0
else
    -- 没有超出 value + 1
    redis.call('INCRBY', key, 1)
    -- 如果 key 中保存的并发计数为 0，说明当前是一个新的时间窗口，它的过期时间设置为窗口的过期时间
    if (current_permits == 0) then
            redis.call('EXPIRE', key, ttl)
	  end
    -- 返回 (放行)
    return 1
end
```

此段 Lua 脚本的逻辑很直观：

1.  通过 `KEYS[1]` 获取传入的 `key` 参数，为某个限流指标的 `key`
2.  通过 `ARGV[1]` 获取传入的 `limit` 参数，为限流值
3.  通过 `ARGV[2]` 获取限流区间 `ttl`
4.  通过 `redis.call`，拿到 `key` 对应的值（默认为 `0`），接着与 `limit` 判断，如果超出表示该被限流；否则，使用 `INCRBY` 增加 `1`，未限流（需要处理初始化的情况，设置 `TTL`）

不过上面代码是有问题的，如果 `key` 之前存在且未设置 `TTL`，那么限速逻辑就会永远生效了（触发 `limit` 值之后），使用时需要注意

#### 令牌桶算法

令牌桶算法也是 Guava 项目中使用的算法，同样采用计算的方式，将时间和 Token 数目联系起来：

```lua
-- key
local key = KEYS[1]
-- 最大存储的令牌数
local max_permits = tonumber(KEYS[2])
-- 每秒钟产生的令牌数
local permits_per_second = tonumber(KEYS[3])
-- 请求的令牌数
local required_permits = tonumber(ARGV[1])

-- 下次请求可以获取令牌的起始时间
local next_free_ticket_micros = tonumber(redis.call('hget', key, 'next_free_ticket_micros') or 0)

-- 当前时间
local time = redis.call('time')
-- time[1] 返回的为秒，time[2] 为 ms
local now_micros = tonumber(time[1]) * 1000000 + tonumber(time[2])

-- 查询获取令牌是否超时（传入参数，单位为 微秒）
if (ARGV[2] ~= nil) then
    -- 获取令牌的超时时间
    local timeout_micros = tonumber(ARGV[2])
    local micros_to_wait = next_free_ticket_micros - now_micros
    if (micros_to_wait> timeout_micros) then
        return micros_to_wait
    end
end

-- 当前存储的令牌数
local stored_permits = tonumber(redis.call('hget', key, 'stored_permits') or 0)
-- 添加令牌的时间间隔（1000000ms 为 1s）
-- 计算生产 1 个令牌需要多少微秒
local stable_interval_micros = 1000000 / permits_per_second

-- 补充令牌
if (now_micros> next_free_ticket_micros) then
    local new_permits = (now_micros - next_free_ticket_micros) / stable_interval_micros
    stored_permits = math.min(max_permits, stored_permits + new_permits)
    -- 补充后，更新下次可以获取令牌的时间
    next_free_ticket_micros = now_micros
end

-- 消耗令牌
local moment_available = next_free_ticket_micros
--  两种情况：required_permits<=stored_permits 或者 required_permits>stored_permits
local stored_permits_to_spend = math.min(required_permits, stored_permits)
local fresh_permits = required_permits - stored_permits_to_spend;
-- 如果 fresh_permits>0，说明令牌桶的剩余数目不够了，需要等待一段时间
local wait_micros = fresh_permits * stable_interval_micros

--  Redis 提供了 redis.replicate_commands() 函数来实现这一功能，把发生数据变更的命令以事务的方式做持久化和主从复制，从而允许在 Lua 脚本内进行随机写入
redis.replicate_commands()
--  存储剩余的令牌数：桶中剩余的数目 - 本次申请的数目
redis.call('hset', key, 'stored_permits', stored_permits - stored_permits_to_spend)
redis.call('hset', key, 'next_free_ticket_micros', next_free_ticket_micros + wait_micros)
redis.call('expire', key, 10)

-- 返回需要等待的时间长度
-- 返回为 0（moment_available==now_micros）表示桶中剩余的令牌足够，不需要等待
return moment_available - now_micros
```

简单分析上上述代码，传入参数为：

- `key`：限流 `key`
- `max_permits`：最大存储的令牌数
- `permits_per_second`：每秒钟产生的令牌数
- `required_permits`：请求的令牌数
- `timeout_micros`：获取令牌的超时时间（非必须）

整个代码的运行流程如下：
![redis-token-bucket-dataflow](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/limiter/redis-token-bucket-dataflow.png)

#### 固定窗口

此算法来自于 [go-redis/redis_rate:v7](https://github.com/go-redis/redis_rate/blob/v7/rate.go#L60) 版本提供的实现，使用 [示例在此](https://github.com/go-redis/redis_rate/blob/v7/README.md)，主要的限速逻辑代码见下：

![redis-rate-fixed-window](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/limiter/redis-rate-window-fixed.png)

在此算法中，使用 `redisPrefix:name-slot` 做固定窗口的标识 key（`allowName` 方法），其中 `slot` 为当前时间戳（秒）除以窗口时延区间值 `udur`；在 `AllowN` 方法实现中，`delay` 的计算出的值是在一个 `dur` 范围内浮动的（由 `utime` 决定）；最终使用 [`INCRBY` 指令](http://redisdoc.com/string/incrby.html) 统计窗口（即 key：`redisPrefix:name-slot`）已使用流量 `count`，并且同时设置 `redisPrefix:name-slot` 在此窗口结束后自动过期

```golang
func (l *Limiter) AllowN(name string, maxn int64, dur time.Duration, n int64,) (count int64, delay time.Duration, allow bool) {
	udur := int64(dur / time.Second)        // 注意，将 duration 转为整数，不可以小于 1s
	utime := time.Now().Unix()
	slot := utime / udur    // 这里的除法有溢出风险
	delay = time.Duration((slot+1)*udur-utime) * time.Second
	if l.Fallback != nil {
		allow = l.Fallback.Allow()
	}
	name = allowName(name, slot)
	count, err := l.incr(name, dur, n)
	if err == nil {
		allow = count <= maxn
	}
	return count, delay, allow
}

//allowName 使用 name+slot 作为 redis 的 key
func allowName(name string, slot int64) string {
	return fmt.Sprintf("%s:%s-%d", redisPrefix, name, slot)
}

//IncrBy+expire 操作合并执行
func (l *Limiter) incr(name string, period time.Duration, n int64) (int64, error) {
    var incr *redis.IntCmd
    // 使用 pipeline 批量操作
	_, err := l.redis.Pipelined(func(pipe redis.Pipeliner) error {
		incr = pipe.IncrBy(name, n)
		pipe.Expire(name, period+30*time.Second)
		return nil
	})

	rate, _ := incr.Result()
	return rate, err
}
```

## 0x03 Go-Redis 提供的分布式限流库

go-redis 官方提供了一个分布式限频库：[Rate limiting for go-redis：redis_rate](https://github.com/go-redis/redis_rate)

#### 使用

```golang
func main() {
        ctx := context.Background()
        rdb := redis.NewClient(&redis.Options{
                Addr: "localhost:6379",
        })
        _ = rdb.FlushDB(ctx).Err()

        limiter := redis_rate.NewLimiter(rdb)
        for {
                res, err := limiter.Allow(ctx, "project:123", redis_rate.PerSecond(5))
                if err != nil {
                        panic(err)
                }
                fmt.Println("allowed", res.Allowed, "remaining", res.Remaining)
                time.Sleep(10 * time.Millisecond)
        }
}
```

#### 限流脚本

redis-rate 提供的 Lua 脚本实现 [在此](https://github.com/go-redis/redis_rate/blob/v9/lua.go)

##  0x04    go-zero 的分布式限流实现
go-zero 提供了两种分布式的限流实现，分别基于 [计数器 periodlimit](https://github.com/zeromicro/go-zero/blob/master/core/limit/periodlimit.go) 和 [令牌桶 tokenlimit](https://github.com/zeromicro/go-zero/blob/master/core/limit/tokenlimit.go)：

**注意：这两种算法的限流区间都是基于秒为单位的。**

| 方案 | 原理 | 适用场景 |
| :-----:|  :----: | :----: |
| periodlimit | 单位时间限制访问次数 | 需要强行限制数据的传输速率 |
| tokenlimit | 令牌桶限流 | 限制数据的平均传输速率，同时允许某种程度的突发传输 |

####    计数器 - periodlimit
该算法是采用 `EXPIRE KEY TTL` 的机制来模拟滑动窗口的限流效果，和上面的计数器机制实现有些类似，利用 `TTL` 做限流窗口的 Span，使用 `INCRBY` 累计做限流值的累加：

![period-limit](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/gozero-tech/period-limiter1.png)

1、限流的核心 lua[逻辑](https://github.com/zeromicro/go-zero/blob/master/core/limit/periodlimit.go#L12)<br>

-   `key[1]`：访问资源的标识
-   `ARGV[1]`：`limit` => 请求总数（QPS），超过则限速
-   `ARGV[2]`：window 大小 => 滑动窗口，用 ttl 模拟出滑动的效果

```lua
--to be compatible with aliyun redis, we cannot use `local key = KEYS[1]` to reuse the key
local limit = tonumber(ARGV[1])
local window = tonumber(ARGV[2])        --window size
local current = redis.call("INCRBY", KEYS[1], 1)
if current == 1 then
    -- 如果是第一次访问，设置过期时间 TTL 为 window size（因为是只限制一段时间的访问次数）
    redis.call("EXPIRE", KEYS[1], window)
end
if current < limit then
    --  Allow
    return 1
elseif current == limit then
    --  hit limiter quota
    return 2
else
    --  may need drop
    return 0
end
```

2、原子性保证 <br>
-   periodlimit 借助 Redis 的 `INCRBY` 做资源访问计数
-   采用 LuaScript 实现整个（窗口）计算，保证计算的原子性


3、缺点 <br>
如果在服务某个时间点有大量并发请求，periodlimit 短期时间内达到 `limit` 阈值，且设置的时间范围 `window` 还远远没有到达。后续请求的处理就成为问题。periodlimit 中并没有处理，而是返回 code，交给了开发者自己处理：
-   如果不做处理，那就是简单的将请求拒绝
-   如果需要处理这些请求，开发者可以借助队列将请求缓冲，减缓请求的压力；或借助于延时队列异步处理


####    令牌桶 - tokenlimit
tokenLimit 相对于 periodlimit 方案，可以允许一部分瞬间的突发流量，桶内存量 token 即可作为流量缓冲区平滑处理突发流量，其工作流程；
-   假设令牌平均生产速率`r`，则每隔`1/r`秒一个令牌被加入到桶中
-   假设桶中最多可以存放`b`个令牌，单位时间按照一定速率匀速的生产 token 放入桶内，直到达到桶容量上限。如果令牌到达时令牌桶已经满了，那么这个令牌会被丢弃
-   处理请求，每次尝试获取一个或多个令牌，如果拿到则处理请求，失败则拒绝请求（限流逻辑）


![token-limiter](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/gozero-tech/token-limiter1.png)

核心的参数如下：
-   `ARGV[1]`：`rate`，表示每秒生成 token 数量，即 token 生成速度
-   `ARGV[2]`：`capacity`，桶容量
-   `ARGV[3]`：`now`，当前请求令牌的时间戳
-   `ARGV[4]`：`requested`，当前请求 token 数量
-   `KEYS[1]`：访问资源的标识
-   `KEYS[2]`：保存上一次访问的刷新时间戳
-   `fill_time`：桶容量 / token 速率，即需要多少单位时间（秒）才能填满桶
-   `ttl`：`ttl` 为填满时间的 2 倍
-   `last_tokens`：当前时刻桶容量

1、限流的 Lua 脚本核心逻辑如下，该原子化的脚本返回 `allowed` 即是否可以活获得预期的 token<br>
```lua
local rate = tonumber(ARGV[1])
local capacity = tonumber(ARGV[2])
local now = tonumber(ARGV[3])
local requested = tonumber(ARGV[4])
local fill_time = capacity/rate
local ttl = math.floor(fill_time*2)

--  桶容量是存储在 KEYS[1] 的 kv 中
local last_tokens = tonumber(redis.call("get", KEYS[1]))
if last_tokens == nil then
    --  如果当前桶容量为 0, 说明是第一次进入, 则默认容量为桶的最大容量
    last_tokens = capacity
end

-- 上一次刷新的时间
local last_refreshed = tonumber(redis.call("get", KEYS[2]))
-- 第一次进入则设置刷新时间为 0
if last_refreshed == nil then
    last_refreshed = 0
end
--  计算距离上次请求的时间跨度（注意这里的单位是：秒）
local delta = math.max(0, now-last_refreshed)
--  距离上次请求的时间跨度, 总共能生产 token 的数量, 如果超多最大容量则丢弃多余的 token
local filled_tokens = math.min(capacity, last_tokens+(delta*rate))
--  计算本次请求 token 数量是否足够，filled_tokens；当前的容量；requested：请求的令牌数
local allowed = filled_tokens >= requested

--  new_tokens：令牌桶剩余数量
local new_tokens = filled_tokens
-- 允许本次 token 申请, 更新剩余数量
if allowed then
    new_tokens = filled_tokens - requested
end
-- 更新剩余 token 数量，注意过期时间是 fill_time 的 2 倍
redis.call("setex", KEYS[1], ttl, new_tokens)
-- 更新刷新时间
redis.call("setex", KEYS[2], ttl, now)
return allowed
```

2、限流器定义 <br>
```golang
type TokenLimiter struct {
    rate int        // 每秒生产速率
    burst int       // 桶容量

    store *redis.Redis
    tokenKey       string       //redis-key
    timestampKey   string       // 桶刷新时间 key
    rescueLock     sync.Mutex
    redisAlive     uint32       // redis 健康标识（atomic）
    rescueLimiter  *xrate.Limiter    // redis 故障时采用进程内令牌桶限流器
    monitorStarted bool    // redis 监控探测任务标识
}
```

3、获取令牌<br>
```golang
func (lim *TokenLimiter) reserveN(now time.Time, n int) bool {
    // 判断redis是否健康
    // redis故障时采用进程内限流器
    // 兜底保障
    if atomic.LoadUint32(&lim.redisAlive) == 0 {
        return lim.rescueLimiter.AllowN(now, n)
    }
    // 执行脚本获取令牌
    resp, err := lim.store.Eval(
        script,
        []string{
            lim.tokenKey,
            lim.timestampKey,
        },
        []string{
            strconv.Itoa(lim.rate),
            strconv.Itoa(lim.burst),
            strconv.FormatInt(now.Unix(), 10),
            strconv.Itoa(n),
        })
    // redis allowed == false
    // Lua boolean false -> r Nil bulk reply
    // 特殊处理key不存在的情况
    if err == redis.Nil {
        return false
    } else if err != nil {
        logx.Errorf("fail to use rate limiter: %s, use in-process limiter for rescue", err)
        // 执行异常，开启redis健康探测任务，然后采用进程内限流器作为兜底
        lim.startMonitor()
        return lim.rescueLimiter.AllowN(now, n)
    }

    code, ok := resp.(int64)
    if !ok {
        logx.Errorf("fail to eval redis script: %v, use in-process limiter for rescue", resp)
        lim.startMonitor()
        return lim.rescueLimiter.AllowN(now, n)
    }

    // redis allowed == true
    // Lua boolean true -> r integer reply with value of 1
    return code == 1
}

// redis健康探测定时任务
func (lim *TokenLimiter) waitForRedis() {
    ticker := time.NewTicker(pingInterval)
    // 健康探测成功时回调此函数
    defer func() {
        ticker.Stop()
        lim.rescueLock.Lock()
        lim.monitorStarted = false
        lim.rescueLock.Unlock()
    }()

    for range ticker.C {
        // ping属于redis内置健康探测命令
        if lim.store.Ping() 
            // 健康探测成功，设置健康标识
            atomic.StoreUint32(&lim.redisAlive, 1)
            return
        }
    }
}
```

上面的代码实现了分布式限流+降级本地限流的逻辑：
![limiter-flow](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/gozero-tech/limiter-flow.png)

####    分布式限流器的降级处理
tokenLimiter 的一个亮点是实现了 redis 故障时的兜底策略，即故障时启动单机版的 ratelimit 做备用限流机制。如果 redis limiter 失效，至少还可以在单服务的 rate limiter 兜底。其降级的[流程图](https://zhuanlan.zhihu.com/p/311320469)如下所示：
![go-zero-limiter](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/gozero-tech/token-limiter2.jpeg)

####    小结
go-zero 中的 tokenlimit 限流方案适用于瞬时流量冲击，现实请求场景并不以恒定的速率。令牌桶相当预请求，当真实的请求到达不至于瞬间被打垮。

##  0x05    总结
本文列举了几种常用的基于Redis的分布式限流的实现，实现的特点为：
1.  采用Redis+Lua脚本实现，保证原子性
2.  常用的方式有令牌桶，计数器等，采用Redis模拟实现
3.  限流支持的时间单位不同，有的精度较高
4.  考虑到Redis的可用性，**在Redis不可用时降级为本地限流器是一种非常好的设计思路**


## 0x06 参考

- [分布式系统限流服务 - Golang&Redis](http://hbchen.com/post/distributed/2019-05-05-rate-limit/)
- [go-redis 的 redis_rate 项目](https://github.com/go-redis/redis_rate/blob/v9/rate.go)
- [基于 window 的 redis 限流实现](https://github.com/hb-go/pkg/blob/e1748b361233dfa970e62f6d296baff0ab00f849/rate/lua/window_rolling.lua)
- [分布式系统高可用实战之限流器（Go 版本实现）](https://cloud.tencent.com/developer/article/1626769)
- [基于 Redis 和 Lua 的分布式限流](http://remcarpediem.net/2019/04/06/%E5%9F%BA%E4%BA%8ERedis%E5%92%8CLua%E7%9A%84%E5%88%86%E5%B8%83%E5%BC%8F%E9%99%90%E6%B5%81/)
- [Redis Best Practices： Basic Rate Limiting](https://redislabs.com/redis-best-practices/basic-rate-limiting/)
- [Rate Limiting Algorithms using Redis](https://medium.com/@SaiRahulAkarapu/rate-limiting-algorithms-using-redis-eb4427b47e33)
- [Go 分布式令牌桶限流 + 兜底策略](https://segmentfault.com/a/1190000041275724)