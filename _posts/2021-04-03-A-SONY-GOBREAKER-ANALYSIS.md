---
layout: post
title: 开源熔断组件分析（一）：gobreaker
subtitle: 分析 Sony 的 gobreaker 熔断器实现（Circuit Breaker 的一种实现）
date: 2021-04-03
header-img: img/super-mario.jpg
author: pandaychen
catalog: true
tags:
  - 熔断
  - Breaker
---

## 0x00 前言

[gobreaker](https://github.com/sony/gobreaker) 实现了 [Circuit Breaker pattern](<https://docs.microsoft.com/en-us/previous-versions/msp-n-p/dn589784(v=pandp.10)?redirectedfrom=MSDN>) 模式的熔断机制。本篇文章简单分析下其实现。

#### Circuit Breaker 回顾

回顾下 Circuit Breaker 的状态机模型：即 `3` 种状态，`4` 种状态（变化）迁移，如下图：

![circuit-breaker](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/breaker/circuit-breaker.png)

Circuit Breaker 状态如下 <br>

- 熔断器关闭状态（StateClosed）, 服务正常访问，不会影响用户请求
- 熔断器开启状态（StateOpen），服务异常，内部服务不可用
- 熔断器半开状态（StateHalfOpen），部分请求限流访问（**恢复阶段**，放行零星请求）

Circuit Breaker 的迁移流程如下：

- 在熔断器关闭状态下，当失败后并满足熔断预设阈值后，将直接转移为熔断器开启状态
- 在熔断器开启状态下，如果过了规定的时间，将进入半开启状态，验证目前服务是否可用
- 在熔断器半开启状态下，如果出现失败，则再次进入熔断器开启状态
- 在熔断器半开启后，所有请求（有限额）都是成功的，则熔断器关闭。关闭后所有请求将正常访问

#### 状态迁移的条件判定

这里再简单列举下状态迁移时的边缘判定条件，需要包含如下几个维度的数据：

- （Breaker 迁移周期内）总请求数：`total_requests`
- 失败请求数：`failed_requests`
- 成功请求数：`succ_requests`
- 熔断器打开到半打开状态的定时器：`timer1`

- 熔断器关闭状态下，有请求到来

  - `total_requests++`
  - 调用成功：`succ_requests++`
  - 调用失败：`failed_requests++`，若失败数（失败率）触发了预设规则（如：最近的故障数在给定的时间段内超过了指定的阈值），则熔断器将迁移到打开状态。同时，Breaker 启动一个超时计时器，当该计时器到期时，熔断器将进入半打开状态
  - 注意：当发生了熔断器状态迁移时，需要清空当前（周期）的计数器

- 熔断器打开状态：来自应用程序的调用立即失败，不统计
- 熔断器半开状态：允许有限数量的请求通过并调用操作，同时统计结果
  - `total_requests++`
  - 调用成功，`succ_requests++`，若成功数（成功率）超过了预设规则，则判断先前的故障已恢复，并且熔断器切换到关闭状态（同时重置计数器）
  - 调用失败，则熔断器将认为故障仍然存在，将熔断器恢复为打开状态，重新设置超时计时器 `timer1`，以使系统有更多时间从故障中恢复

## 0x01 gobreaker 的实现 - 基础结构

完整的代码分析 [在此](https://github.com/pandaychen/gobreaker)。按照 [熔断器的转换流程](https://pandaychen.github.io/2020/04/01/MICROSERVICE-BREAKER-INTRO/#0x03 - 熔断器的转换流程) 来分析 gobreaker 实现。

#### 熔断器状态

```golang
const (
	StateClosed State = iota		// 熔断器关闭 0
	StateHalfOpen					// 半开放
	StateOpen						// 打开
)
```

####	熔断器状态统计
如何判定是否要进行状态迁移呢？需要有个全局的统计结构 `Counts`：

```golang
// Counts holds the numbers of requests and their successes/failures.
// CircuitBreaker clears the internal Counts either
// on the change of the state or at the closed-state intervals.
// Counts ignores the results of the requests sent before clearing.
type Counts struct {
	Requests             uint32 // 请求次数
    TotalSuccesses       uint32 // 总共成功次数
    TotalFailures        uint32 // 总共失败次数
    ConsecutiveSuccesses uint32 // 连续成功次数
    ConsecutiveFailures  uint32 // 连续失败次数
}

func (c *Counts) onRequest() {
	c.Requests++
}

func (c *Counts) onSuccess() {
	c.TotalSuccesses++
	c.ConsecutiveSuccesses++
	c.ConsecutiveFailures = 0
}

func (c *Counts) onFailure() {
	c.TotalFailures++
	c.ConsecutiveFailures++
	c.ConsecutiveSuccesses = 0
}

func (c *Counts) clear() {
	c.Requests = 0
	c.TotalSuccesses = 0
	c.TotalFailures = 0
	c.ConsecutiveSuccesses = 0
	c.ConsecutiveFailures = 0
}
```

注意，`Counts` 的生效周期仅限于一个 `Generation` 周期内，有点类似于限流器的固定窗口机制。

####	统计周期 generation
```golang
//toNewGeneration: 生成新的 generation。 主要是清空 counts 和设置 expiry（过期时间）
//1. 当状态为 Closed 时 expiry 为 Closed 的过期时间（当前时间 + interval）
//2. 当状态为 Open 时 expiry 为 Open 的过期时间（当前时间 + timeout）

func (cb *CircuitBreaker) toNewGeneration(now time.Time) {
	cb.generation++
	// 清空单个周期内的计数结构
	cb.counts.clear()

	var zero time.Time
	switch cb.state {
	// 当熔断器在 CLOSE 状态下
	case StateClosed:
		if cb.interval == 0 {
			//defaultInterval
			cb.expiry = zero
		} else {
			//
			cb.expiry = now.Add(cb.interval)
		}
	case StateOpen:
		cb.expiry = now.Add(cb.timeout)
	default: // StateHalfOpen
		cb.expiry = zero
	}
}
```

##	0x02	gobreaker 的代码分析

####	熔断器外部接口
熔断器的执行 `Execute` 方法包含三个部分，参数 `req` 为封装的业务方法。该函数分为三步：
1.	`beforeRequest`：请求之前的判定（如：如果熔断器处于打开状态就直接返回）
2.	执行请求：服务的请求执行
3.	`afterRequest`：请求后的状态和计数的更新

```golang
// 核心执行函数 Execute： 该函数分为三步 beforeRequest、 执行请求、 afterRequest
func (cb *CircuitBreaker) Execute(req func() (interface{}, error)) (interface{}, error) {
	generation, err := cb.beforeRequest()
	if err != nil {
		return nil, err
	}

	defer func() {
		e := recover()
		if e != nil {
			cb.afterRequest(generation, false)
			panic(e) //if panic，继续 panic 给上层调用者去 recover，有趣
		}
	}()

	// 执行真正的用户调用
	result, err := req()

	// 调用后更新熔断器状态
	cb.afterRequest(generation, cb.isSuccessful(err))
	return result, err
}
```

####	步骤 1: beforeRequest
`beforeRequest` 的作用是判断是否放行请求，计数或达到切换新条件刚切换，主要功能都在 `currentState` 方法中；
1.	判断是否 `Closed`，如是，放行所有请求；并且判断时间是否达到 `Interval` 周期，从而清空计数，调用 `toNewGeneration` 进入新周期（清空计数）；
2.	如果是 `Open` 状态，返回 `ErrOpenState`，—不放行所有请求；同样判断周期时间，到达则同样调用 `toNewGeneration` 进入新周期
3.	如果是 `HalfOpen` 状态，则判断是否已放行 `MaxRequests` 个请求，如未达到则放行；否则返回: `ErrTooManyRequests`，此函数一旦放行请求，就调用 `conut.onRequest` 对请求计数加 `1`，进入请求逻辑

```golang
/*
beforeRequest 函数的核心功能：判断是否放行请求，计数或达到切换新条件刚切换。
1. 判断是否 Closed，如是，放行所有请求。
	-- 并且判断时间是否达到 Interval 周期，从而清空计数，进入新周期，调用 toNewGeneration()
2. 如果是 Open 状态，返回 ErrOpenState，不放行所有请求。
	-- 同样判断周期时间，到达则 同样调用 toNewGeneration()，清空计数
3. 如果是 half-open 状态，则判断是否已放行 MaxRequests 个请求，如未达到刚放行；否则返回: ErrTooManyRequests。
4. 此函数一旦放行请求，就会对请求计数加 1（conut.onRequest())，请求后到另一个关键函数 : afterRequest()。
*/
func (cb *CircuitBreaker) beforeRequest() (uint64, error) {
	cb.mutex.Lock()
	defer cb.mutex.Unlock()

	now := time.Now()
	// 获取当前熔断器的状态和 generation
	state, generation := cb.currentState(now)

	if state == StateOpen {
		// 如果熔断器处于打开状态，禁止请求，直接返回错误
		return generation, ErrOpenState
	} else if state == StateHalfOpen && cb.counts.Requests >= cb.maxRequests {
		//half-open 状态 && 请求超量，也拒绝请求
		return generation, ErrTooManyRequests
	}

	// 其他情况，放行请求，走到 afterRequest 逻辑
	cb.counts.onRequest()
	return generation, nil
}
```

####	步骤 2：req
`req`方法: 获取请求结果，注意这里做了`defer`，防止在`req()`执行中异常退出。

```golang
// 核心执行函数 Execute： 该函数分为三步 beforeRequest、 执行请求、 afterRequest
func (cb *CircuitBreaker) Execute(req func() (interface{}, error)) (interface{}, error) {
	//...
	defer func() {
		e := recover()
		if e != nil {
			cb.afterRequest(generation, false)
			panic(e) //if panic，继续 panic 给上层调用者去 recover，有趣
		}
	}()

	// 执行真正的用户调用
	result, err := req()

	//...
}
```


####	步骤 3：afterRequest
`afterRequest` 的作用是上一步业务请求的结果进行对成功 / 失败进行计数，达到条件则切换状态；注意参数 `cb.afterRequest(generation, cb.isSuccessful(err))` 中 `isSuccessful`，默认只是判断是否为 `nil`，在实际中这样是有问题的，**我们需要指定进行熔断失败计数的错误类型**，比如服务调用超时，服务不可达等，其他业务逻辑错误不应该作为熔断器失败的计数统计条件；

`afterRequest` 的逻辑是
1.	调用公共函数 `currentState(now)`，先判断是否进入一个新的计数时间周期 `Interval`, 是则重置计数，改变熔断器状态，并返回新周期的状态
2.	熔断状态机计数统计更新

```golang
/*
函数核心内容很简单，就对成功 / 失败进行计数，达到条件则切换状态。
与 beforeRequest 一样，会调用公共函数 currentState(now)
currentState(now) 先判断是否进入一个先的计数时间周期 (Interval), 是则重置计数，改变熔断器状态，并返回新一代。
如果 request 耗时大于 Interval, 几本每次都会进入新的计数周期，熔断器就没什么意义了
*/
func (cb *CircuitBreaker) afterRequest(before uint64, success bool) {
	cb.mutex.Lock()
	defer cb.mutex.Unlock()

	now := time.Now()
	state, generation := cb.currentState(now)
	if generation != before {
		// 说明，在 currentState 已经更新了代数，直接返回吧
		return
	}

	// 否则，说明还在同一代中，根据 err（是否为 nil，这里比较简单）更新计数
	if success {
		// 更新 succ 技数
		cb.onSuccess(state, now)
	} else {
		// 更新错误计数
		cb.onFailure(state, now)
	}
}
```

####	当前状态的判定及迁移
`currentState` 方法承担了状态机的迁移工作。下一（阶段）状态的计算，是依据当前状态来的：
-	如果当前状态为关闭 `StateClosed`，则通过周期判断 `toNewGeneration` 是否进入下一个新周期（重置 `Couters`）
-	如果当前状态为已开启 `StateOpen`，则判断是否已经超时，超时就可以变更状态到半开；如果当前状态为关闭状态，则通过周期判断是否进入下一个周期
-	如果当前状态为半开启 `StateHalfOpen`，则直接返回（后续根据结果计算再决定是否进行状态变迁）

```golang
func (cb *CircuitBreaker) currentState(now time.Time) (State, uint64) {
  switch cb.state {
  case StateClosed:
    if !cb.expiry.IsZero() && cb.expiry.Before(now) { // 是否需要进入下一个计数周期
      cb.toNewGeneration(now)
    }
  case StateOpen:
    if cb.expiry.Before(now) {
      // 熔断器由开启变更为半开
      cb.setState(StateHalfOpen, now)
    }
  }
  return cb.state, cb.generation
}
```

####	迁移状态 setState
`setState` 方法是用来执行熔断状态机迁移的，每当设置新状态时，需要重置当前的 `Generation`：
```golang
// 设置当前熔断器状态
func (cb *CircuitBreaker) setState(state State, now time.Time) {
	if cb.state == state {
		// 无需设置
		return
	}

	prev := cb.state
	cb.state = state
	// 每当设置新状态时，需要重置当前的 generation
	cb.toNewGeneration(now)

	// 如果用户设置了状态变迁回调，那么就调用
	if cb.onStateChange != nil {
		cb.onStateChange(cb.name, prev, state)
	}
}
```


####	在何处迁移状态
所以，参照前文 [微服务基础之熔断保护（Breaker）](https://pandaychen.github.io/2020/04/01/MICROSERVICE-BREAKER-INTRO/#0x03 - 熔断器的转换流程) 熔断状态机迁移的状态机，有下面几处调用了状态迁移方法：

1、动作 `B`：当熔断器从关闭状态到打开状态时，由每次熔断器调用 `Execute` 中的 `afterRequest` 方法来检查并设置 <br>
```golang
func (cb *CircuitBreaker) afterRequest(before uint64, success bool) {
	cb.mutex.Lock()
	defer cb.mutex.Unlock()

	now := time.Now()
	state, generation := cb.currentState(now)
	if generation != before {
		// 说明，在 currentState 已经更新了代数，直接返回吧
		return
	}

	// 否则，说明还在同一代中，根据 err（是否为 nil，这里比较简单）更新计数
	if success {
		// 更新 succ
		cb.onSuccess(state, now)
	} else {
		cb.onFailure(state, now)
	}
}

// 调用失败情况下的处理
func (cb *CircuitBreaker) onFailure(state State, now time.Time) {
	switch state {
	case StateClosed:
		cb.counts.onFailure() // 失败计数 ++
		if cb.readyToTrip(cb.counts) {
			// 调用触发熔断器由关闭 => 打开的判断方法（可由用户传入，默认方法 defaultReadyToTrip 是连续的错误次数 > 5）
			// 设置熔断器为打开状态
			cb.setState(StateOpen, now)
		}
	case StateHalfOpen:
		// 在 half-open 情况下，如果仍然调用失败，那么继续把熔断器设置为打开状态
		cb.setState(StateOpen, now)
	}
}
```

gobreaker 的默认逻辑是，在 `StateClosed` 关闭状态下， 当连续失败次数 `>5` 次时， 则切换到 `StateOpen` 打开状态，用户可以指定回调函数

2、动作 `D`，熔断器从打开到半打开状态的迁移，在每个 `generation` 中都会触发的 `currentState` 方法中完成（基于时间跨度参数）<br>
```golang
//currentState: 获取当前状态
//1、当 Closed 时且 expiry 过期，调用 toNewGeneration 生成新的 generation
//2、当 Open 时且 expiry 过期，设为 halfOpen
func (cb *CircuitBreaker) currentState(now time.Time) (State, uint64) {
	switch cb.state {
	// 熔断器关闭时
	case StateClosed:
		if !cb.expiry.IsZero() /*cb.expiry 非 0 值 */ && cb.expiry.Before(now) /*cb.expiry 比 now 早，说明 cb.expiry 过期 */ {
			// 需要重新生成一个周期
			cb.toNewGeneration(now)
		}
		// 否则不需要
	case StateOpen:
		// 熔断器打开时
		if cb.expiry.Before(now) {
			// 如果打开时，cb.expiry 过期，那么熔断器需要进入 half-open 状态
			// 注意：在此来完成从熔断器打开 => 熔断器半打开的触发逻辑！！！！！
			cb.setState(StateHalfOpen, now)
		}
	}
	return cb.state, cb.generation
}
```
在 `StateOpen` 状态，受参数 `Settings.Timeout` 控制，变迁到 `StateHalfOpen` 半开放状态

3、动作 `F`，在 `StateHalfOpen` 半开放状态时，如果进行请求（探测）仍然失败，则继续切换到 `StateOpen` 熔断打开状态。逻辑在前面的 `onFailure` 方法中 <br>

4、动作 `G`，在 `StateHalfOpen` 半开放状态时，如果（当前这代 counts 中）连续 succ 的数目 `cb.counts.ConsecutiveSuccesses` 超过 `Settings.MaxRequests`，那么则重置当前熔断器的状态为 closed<br>
```golang
func (cb *CircuitBreaker) onSuccess(state State, now time.Time) {
	switch state {
	case StateClosed:
		cb.counts.onSuccess()
	case StateHalfOpen:
		// 在 half-open 状态下，如果（当前这代 counts 中）连续 succ 的数目超过 maxRequests，那么则重置当前熔断器的状态为 closed（关闭）
		cb.counts.onSuccess()
		if cb.counts.ConsecutiveSuccesses >= cb.maxRequests {
			cb.setState(StateClosed, now)
		}
		// 这里不可能出现 stateOpen 状态
	}
}
```

##	0x03	总结
gobreaker 的实现并不难理解，它使用了一个 `Generation` 计数周期的概念，每一个时间周期 `Interval` 的计数 `count` 状态称为一个 `Generation`。在 `beforeRequest`/`afterRequest` 的两个方法中，实现了两个状态自动切换的机制：
-	在同一个 `Generation` 周期内，计数满足状态切换条件，即自动切换
-	超过一个 `Generation` 时间周期的也会自动切换
gobreaker 并没有使用定时器，只在请求调用时，去检测当时状态与时间间隔来切换，这也是比较典型的技巧。此外，在 `beforeRequest`/`afterRequest` 中都强制加了[互斥锁](https://github.com/pandaychen/gobreaker/blob/master/gobreaker.go#L314)来确保并发安全

gobreaker 的缺点如下：
1.	固定的时间窗口（相较于滑动窗口精准度欠缺），如果业务请求 `Request` 耗时大于 `Interval`, 那么每次都会进入新的计数周期，熔断器就没什么意义了（即如果用户请求时间过长，请求前后状态已改变，则对此次请求不进行计数）
2.	在半开放状态，没有提供回调方法来自定义状态变迁
3.	gobreaker 的各类参数都需要用户自行传入，配置的准确性还需要多次测试才能得出

## 0x04	参考

- [gobreaker.go 实现](https://github.com/sony/gobreaker/blob/master/gobreaker.go)
- [gobreaker 简单注释](https://github.com/pandaychen/gobreaker)
- [sre_breaker.go](https://github.com/go-kratos/kratos/blob/master/pkg/net/netutil/breaker/sre_breaker.go)
