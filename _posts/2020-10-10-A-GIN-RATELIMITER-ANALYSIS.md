---
layout:     post
title:      在 Golang-Gin 框架中集成 Ratelimiter 限流中间件
subtitle:   Gin 实践：如何实现限流中间件
date:       2020-10-10
header-img: img/super-mario.jpg
author:     pandaychen
catalog:    true
tags:
    - 限流
---

##  0x00    前言
本篇文章介绍下如何在 gin 中实现限速的中间件。限速通常是限定服务的 QPS 或者限制并发请求量、连接数或业务支撑的关键指标。

##  0x01    Channel 方式实现
这里有个使用 channel 实现的 [gin-limiter](https://github.com/aviddiviner/gin-limit) 中间件，通过 `sem := make(chan struct{}, n)` 的操作来实现并发控制，核心逻辑如下：
```golang
func MaxAllowed(n int) gin.HandlerFunc {
    // 使用定长 channel，利用 channel 跨协程阻塞获取的能力
	sem := make(chan struct{}, n)
	acquire := func() { sem <- struct{}{} }
	release := func() { <-sem}
	return func(c *gin.Context) {
		acquire() // before request
		defer release() // after request
		c.Next()
	}
}
```

从实现来看，此限速逻辑只是限制并发数，在执行真正 HTTP 业务逻辑之前，在此中间件中判断是否达到最大并发数，如果达到并发限制，相关的请求就会阻塞等待在 `acquire()` 上。极端场景下，当过载发生时，用户请求只是被阻塞住，等 channel 可读时，仍然会执行业务逻辑，起不到限流保护后端的作用。

##  0x03    优化：对限速的部分超时
我们来对上面的代码做一些优化：
1.	使用 `select` 优化 token 获取，增加超时的 case 分支，当超时后仍然拿不到 token，就对原始请求返回 `504` 超时错误
2.	考虑到业务 Panic 场景下的 recover 机制
3.	考虑到多个中间件执行时能正确回收 channel

```golang
func MaxAllowed(max int, timeoutMs int) gin.HandlerFunc {
	// 初始化 sem
	sem := make(chan struct{}, max)
	return func(c *gin.Context) {
		var called, fulled bool
		defer func() {
			if called == false && fulled == false { // 可能其他中间件先执行或后执行，所以这里要加一下判断，进行 channel 回收
				<-sem
			}

			if err := recover(); err != nil {

			}
		}()

		select {
		case sem <- struct{}{}:
			// 未达到限速并发上限，获取 token
			c.Next()	// 转入其他中间件逻辑

			// 关键逻辑
			called = true // 如果其他中间件提前捕获 panic，下面代码还是会被执行
			<-sem
		case <-time.After(time.Duration(timeoutMs) * time.Millisecond):
			// 达到并发上限，且等待 timeoutMs 毫秒仍然未获取票据（说明其他请求未归还 token）
			fulled = true
			// 注意：不能直接 return，否则其他中间件仍然会执行，需要使用 Abort 直接返回错误
			//504 Gateway Timeout
			c.AbortWithStatus(504)
		}
	}
}
```

##	0x04	令牌 / 漏桶方式实现
前面介绍了使用 channel 方式的实现，这里介绍下使用 [令牌桶的方式](https://github.com/juju/ratelimit) 实现中间件。此外中间件的生效范围可以灵活指定：<br>
-	需要对全站限流，可以注册为全局的中间件
-	需要某一组路由（Group）需要限流，那么就只需将该限流中间件注册到对应的路由组即可
-	需要对某个指定的 cgi 需要限流，那么将中间件初始化为一个 `map[string]*ratelimit.Bucket` 的 key（以 `cgipath` 为 key），限速的时候先根据 `cgipath` 获取到限速策略，然后再执行 `bucket.TakeAvailable()` 方法即可

下面简单给出全局限速的实现代码：
```golang
func MaxAllowed(fillInterval time.Duration, cap int64) func(c *gin.Context) {
	// 初始化全局 / 局部的 bucket 策略
	bucket := ratelimit.NewBucket(fillInterval, cap)

	// 返回限流逻辑
	return func(c *gin.Context) {
		// 如果取不到令牌就中断本次请求返回 rate limit...
		if bucket.TakeAvailable(1) < 1 {
			c.String(http.StatusOK, "Rate Limit,Drop")
			c.Abort()
			// 或者直接 c.AbortWithStatus(504)
			return
		}
		c.Next()
	}
}
```

##  0x05	参考
-   [Golang 标准库限流器 time/rate 实现剖析](https://www.cyhone.com/articles/analisys-of-golang-rate/)
-	[gin 框架中使用限流中间件](https://www.liwenzhou.com/posts/Go/ratelimit/)