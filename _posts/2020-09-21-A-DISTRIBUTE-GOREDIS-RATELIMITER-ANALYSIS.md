---
layout:     post
title:      分布式限流：基于 Redis 实现
subtitle:   基于 Redis 实现的分布式限流方案总结
date:       2020-11-21
author:     pandaychen
catalog:    true
tags:
	- Redis
	- 限流
	- 分布式
---

##  0x00	前言
本文梳理下基于 Redis 实现的 分布式限流的策略和算法。这里主要关注 "分布式" 的场景，主要的实现算法也无非以下几种：
-	计数器
-	固定窗口
-	滑动窗口
-	漏桶
-	令牌桶

##	0x01	Why Redis && Lua？
分布式限流本质上是一个集群并发问题，Redis + Lua 的方案非常适合此场景：
1.	Redis 单线程特性，适合解决分布式集群的并发问题
2.	Redis 本身支持 Lua 脚本执行，可以实现原子执行的效果

Redis 执行 Lua 脚本会以原子性方式进行，在执行脚本时不会再执行其他脚本或命令。并且，Redis 只要开始执行 Lua 脚本，就会一直执行完该脚本再进行其他操作，所以 Lua 脚本中 <font color="#dd0000"> 不能进行耗时操作 </font>，Redis + Lua 的应用场景非常多，如分布式锁，限流，秒杀等等。

使用 Redis + Lua 方案有如下注意事项：
1.  使用 Lua 脚本实现原子性操作的 CAS，避免不同客户端先读 Redis 数据，经过计算后再写数据造成的并发问题
2.  前后多次请求的结果有依赖关系时，最好使用 Lua 脚本将多个请求整合为一个；但请求前后无依赖时，使用 pipeline 方式，比 Lua 脚本方便
3.  为了保证安全性，在 Lua 脚本中不要定义自己的全局变量，以免污染 Redis 内嵌的 Lua 环境。因为 Lua 脚本中你会使用一些预制的全局变量，比如说 `redis.call()`
4.  注意 Lua 脚本的时间复杂度，Redis 的单线程同样会阻塞在 Lua 脚本的执行中，Lua 脚本不要进行高耗时操作
5.  Redis 要求单个 Lua 脚本操作的 key 必须在同一个 Redis 节点上，因此 Redis Cluster 方式需要设置 HashTag

##	0x02	常用方法
本小节介绍下常用的分布式限流方案。

####	计数器
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
1.  通过 `KEYS[1]` 获取传入的 key 参数，为某个限流指标的 key
2.  通过 `ARGV[1]` 获取传入的 limit 参数，为限流值
3.  通过 `ARGV[2]` 获取限流区间 ttl
4.  通过 `redis.call`，拿到 key 对应的值（默认为 0），接着与 limit 判断，如果超出表示该被限流；否则，使用 `INCRBY` 增加 1，未限流（需要处理初始化的情况，设置 `TTL`）

不过上面代码是有问题的，如果 key 之前存在且未设置 `TTL`，那么限速逻辑就会永远生效了（触发 limit 值之后），使用时需要注意

####	令牌桶算法
令牌桶算法也是 Guava 中使用的算法，同样采用计算的方式，将时间和 Token 数目联系起来：
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

-- 查询获取令牌是否超时（传入参数，单位为 ms）
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
local stable_interval_micros = 1000000 / permits_per_second

-- 补充令牌
if (now_micros> next_free_ticket_micros) then
    local new_permits = (now_micros - next_free_ticket_micros) / stable_interval_micros
    stored_permits = math.min(max_permits, stored_permits + new_permits)
    next_free_ticket_micros = now_micros
end

-- 消耗令牌
local moment_available = next_free_ticket_micros
local stored_permits_to_spend = math.min(required_permits, stored_permits)
local fresh_permits = required_permits - stored_permits_to_spend;
local wait_micros = fresh_permits * stable_interval_micros

--  Redis 提供了 redis.replicate_commands() 函数来实现这一功能，把发生数据变更的命令以事务的方式做持久化和主从复制，从而允许在 Lua 脚本内进行随机写入
redis.replicate_commands()
redis.call('hset', key, 'stored_permits', stored_permits - stored_permits_to_spend)
redis.call('hset', key, 'next_free_ticket_micros', next_free_ticket_micros + wait_micros)
redis.call('expire', key, 10)

-- 返回需要等待的时间长度
return moment_available - now_micros
```

简单分析上上述代码，传入参数为：
- key：限流 key

##  0x0 Go-Redis 提供的分布式限流库
go-redis 官方提供了一个分布式限频库：[Rate limiting for go-redis：redis_rate](https://github.com/go-redis/redis_rate)

####  使用
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

####  限流脚本
redis-rate 提供的 Lua 脚本实现 [在此](https://github.com/go-redis/redis_rate/blob/v9/lua.go)


##  参考
-   [](http://hbchen.com/post/distributed/2019-05-05-rate-limit/)
-   [](https://github.com/go-redis/redis_rate/blob/v9/rate.go)
-   [](https://github.com/hb-go/pkg/blob/e1748b361233dfa970e62f6d296baff0ab00f849/rate/lua/window_rolling.lua)
-   [](https://github.com/hb-go/pkg/blob/e1748b361233dfa970e62f6d296baff0ab00f849/rate/window_rolling.go)
-   [](https://cloud.tencent.com/developer/article/1626769)
-	  [基于 Redis 和 Lua 的分布式限流](http://remcarpediem.net/2019/04/06/%E5%9F%BA%E4%BA%8ERedis%E5%92%8CLua%E7%9A%84%E5%88%86%E5%B8%83%E5%BC%8F%E9%99%90%E6%B5%81/)
-	  [Redis Best Practices：	Basic Rate Limiting](https://redislabs.com/redis-best-practices/basic-rate-limiting/)
-	  [](https://www.ctolib.com/vearne-ratelimit.html)
-	  [](https://medium.com/@SaiRahulAkarapu/rate-limiting-algorithms-using-redis-eb4427b47e33)