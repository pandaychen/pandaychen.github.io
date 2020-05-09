---
layout:     post
title:      微服务基础之熔断保护（Breaker）
subtitle:
date:       2020-04-01
author:     pandaychen
header-img:
catalog: true
category:   false
tags:
    - 熔断
---


##  0x00    前言

什么是熔断器 / Breaker？

&emsp;&emsp; 熔断器是为了当依赖的服务已经出现故障时，主动阻止对依赖服务的请求（通常情况下是执行本地的服务降级办法，亦或直接返回错误），从而保证自身服务的正常运行不受依赖服务影响，防止雪崩效应。

在实现上，它的思路是在调用方 Caller 增加一种 "避让" 机制，当下游出现异常时能够停止（熔断）对下游的继续请求，当等待一段时间后缓慢放行部分的调用流量，并当这部分流量依旧正常的情况下，解除 "熔断" 状态。当下游再次出现异常时，再次打开，周而复始。

一般而言，熔断保护策略部署在服务端保护策略的最后一级，在限流之后。

##  0x01	原理
熔断器的本质就是状态机，包含了熔断检测、熔断关闭、数据统计三个模块。如下图状态机中三种状态的变迁：

OPEN ------ HALFOPEN ------ CLOSED

-	OPEN 状态：熔断器打开，使用快速失败返回，调用链结束

-	HALFOPEN 状态：当熔断开启一段时间后，尝试阶段（熔断器打开，但允许放过少部分请求）

-	CLOSED 状态：熔断器关闭，正常调用

![image](https://s1.ax1x.com/2020/04/24/J0JYb8.png)

##	0x02	熔断算法指标的量化

![image](https://s1.ax1x.com/2020/04/23/J093dg.png)

当 Service-E 服务出现故障时，Service-B 的熔断检测模块，** 主动 ** 检测到 Client 调用 Service-E 服务错误率达到设置阈值，从而 ** 主动 ** 开启熔断，开启熔断的结果，是访问 Service-E 的请求全部返回错误，或者按照默认值处理；当 Service-E 服务恢复时，** 自动 ** 关闭熔断状态。这里的好处：
-   保护了 Service-B 自身的稳定性
-   降低对 Service-E 的透传请求，防止服务链路上引发雪崩效应

在上面的描述中，两个核心点：开启熔断和关闭熔断的策略。针对这两种场景，在编码中需要量化的概念：

熔断检测模块：检测失败率是否超过阈值，开启熔断
-   开启熔断（熔断的阀值量化）：服务错误率
	-   统计区间的总请求数、请求失败数（目标服务调用延迟较大、超时、调用失败都可以作为失败统计指标，要排除掉逻辑错误）
	-	统计方式：一般采用滑动窗口进行统计（避免毛刺现象）
-	熔断判断条件
	-	时间维度：多长时间内的超时请求达到多少，触发熔断
	-	请求维度：多长时间内的错误（超时）请求达到多少，触发熔断

熔断关闭模块：主动探测依赖服务
-	探测服务恢复：如何量化情况好转：多长时间之后超时请求数低于多少关闭熔断
-	关闭熔断：情况好转，恢复目标服务调用

##	0x03	熔断器的应用方式
熔断器的使用主体一般是客户端侧（也可以是服务侧中的客户端调用），比如在 Kratos 中，默认集成熔断器功能的组件有：
-	RPC client: [pkg/net/rpc/warden/client](https://github.com/go-kratos/kratos/blob/master/pkg/net/rpc/warden/client.go)
-	Mysql client：pkg/database/sql
-	Tidb client：pkg/database/tidb
-	Http client：pkg/net/http/blademaster

熔断器的应用一般离不开下面几个步骤：
1.	初始化熔断器（组），主要包含了开启熔断的配置（错误率、时长、探测因素等）
2.	客户端在真正发起请求调用（如 RPC、网络 IO）前，判断熔断器 Breaker（的状态机）是否处于开启 / 关闭状态
3.	连接执行成功或失败将结果上报熔断器 Breaker，Kratos 中是直接将结果更新到滑动窗口 `RollingCouter` 中，作为 `Allow()` 的计算依据
4.	熔断器 Breaker 内部的状态机实时计算结果（根据请求成功情况），实时更新状态机当前状态

Kratos 文档给出了使用 breaker 的步骤：
```golang
// Breaker 配置说明
type Config struct {
	SwitchOff bool // 熔断器开关, 默认关 false.

	K float64  // 触发熔断的错误率（K = 1 - 1 / 错误率）

	Window  xtime.Duration // 统计桶窗口时间
	Bucket  int  // 统计桶大小
	Request int64 // 触发熔断的最少请求数量（请求少于该值时不会触发熔断）
}

......

// 初始化熔断器组
// 一组熔断器公用同一个配置项，可从分组内取出单个熔断器使用。可用在比如 mysql 主从分离等场景。
brkGroup := breaker.NewGroup(&breaker.Config{})
// 为每一个连接指定一个 brekaker
// 此处假设一个客户端连接对象实例为 conn
//breakName 定义熔断器名称 一般可以使用连接地址
breakName = conn.Addr
conn.breaker = brkGroup.Get(breakName)

// 在连接发出请求前判断熔断器状态
if err = conn.breaker.Allow(); err != nil {
	return
}

// 连接执行成功或失败将结果告知 braker
if(respErr != nil){
	conn.breaker.MarkFailed()
}else{
	conn.breaker.MarkSuccess()
}
```

####	mysqlclient 中的使用
这里以 Kratos 中 mysqlclient 的 [exec 方法](https://github.com/go-kratos/kratos/blob/master/pkg/database/sql/sql.go#L300) 为例，说明下上面的步骤：
```golang
func (db *conn) exec(c context.Context, query string, args ...interface{}) (res sql.Result, err error) {
	now := time.Now()
	defer slowLog(fmt.Sprintf("Exec query(%s) args(%+v)", query, args), now)
	if t, ok := trace.FromContext(c); ok {
		t = t.Fork(_family, "exec")
		t.SetTag(trace.String(trace.TagAddress, db.addr), trace.String(trace.TagComment, query))
		defer t.Finish(&err)
	}
	// 真正发起调用前判断是否需要熔断
	if err = db.breaker.Allow(); err != nil {
		_metricReqErr.Inc(db.addr, db.addr, "exec", "breaker")
		return
	}
	_, c, cancel := db.conf.ExecTimeout.Shrink(c)
	res, err = db.ExecContext(c, query, args...)
	cancel()
	// 上报熔断结果
	db.onBreaker(&err)
	_metricReqDur.Observe(int64(time.Since(now)/time.Millisecond), db.addr, db.addr, "exec")
	if err != nil {
		err = errors.Wrapf(err, "exec:%s, args:%+v", query, args)
	}
	return
}
```

在 `db.onBreaker(&err)` 中，根据 `err` 的类型来向熔断器 Breaker 上报成功或失败状态：
```golang
func (db *conn) onBreaker(err *error) {
	if err != nil && *err != nil && *err != sql.ErrNoRows && *err != sql.ErrTxDone {
		db.breaker.MarkFailed()
	} else {
		db.breaker.MarkSuccess()
	}
}
```

熟悉 `RollingCounter` 的接口可知，`MarkSuccess` 是成功 `+1`、总数 `+1`；`MarkFailed` 是成功 `+0`、总数 `+1`：
```golang
func (b *sreBreaker) MarkSuccess() {
	b.stat.Add(1)
}

func (b *sreBreaker) MarkFailed() {
	// NOTE: when client reject requets locally, continue add counter let the
	// drop ratio higher.
	b.stat.Add(0)
}
```



##	0x04	总结
本文介绍了微服务中常用熔断机制的原理，熔断机制是预防服务雪崩的最有效的一种手段。目前在 gRPC 项目中，就使用了 [Hystrix-Go](https://github.com/afex/hystrix-go) 作为客户端的熔断实现，当然 Kratos 中的自适应熔断算法（基于 Google-SRE 设计）的现网应用效果可能会更优雅。
下一篇文章来分析下 Hystrix-Go 是如何实现熔断策略的。

##  0x05	参考
-   [微服务 - 熔断机制](http://blog.zhuxingsheng.com/blog/micro-service-fuse-mechanism.html)
-   [CircuitBreaker](https://martinfowler.com/bliki/CircuitBreaker.html)
-	[Kratos-Breaker 文档](https://github.com/go-kratos/kratos/blob/master/doc/wiki-cn/breaker.md)
-	[Golang 中使用断路器](https://yangxikun.com/golang/2019/08/10/golang-circuit.html)

转载请注明出处，本文采用 [CC4.0](http://creativecommons.org/licenses/by-nc-nd/4.0/) 协议授权