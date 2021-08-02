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

- 熔断器关闭状态（StateClosed）, 服务正常访问
- 熔断器开启状态（StateOpen），服务异常
- 熔断器半开状态（StateHalfOpen），部分请求限流访问

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

## 0x01 gobreaker 的实现

#### 熔断器状态定义及统计

```golang
const (
	StateClosed State = iota
	StateHalfOpen
	StateOpen
)
```

如何判定是否要进行状态迁移呢？需要有个全局的统计结构：

```golang
// Counts holds the numbers of requests and their successes/failures.
// CircuitBreaker clears the internal Counts either
// on the change of the state or at the closed-state intervals.
// Counts ignores the results of the requests sent before clearing.
type Counts struct {
	Requests             uint32
	TotalSuccesses       uint32
	TotalFailures        uint32
	ConsecutiveSuccesses uint32
	ConsecutiveFailures  uint32
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

## 参考

- [gobreaker.go 实现](https://github.com/sony/gobreaker/blob/master/gobreaker.go)
- [gobreaker 简单注释](https://github.com/pandaychen/gobreaker)
- [sre_breaker.go](https://github.com/go-kratos/kratos/blob/master/pkg/net/netutil/breaker/sre_breaker.go)
