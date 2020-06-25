---
layout:     post
title:      Kratos 源码分析：Database 之 Mysql 的封装
subtitle:   分析 Kratos 的数据库 MYSQL-API
date:       2020-07-10
header-img: img/golang-tools-fun.png
author:     pandaychen
catalog:    true
tags:
    - Kratos
---

## 0x00	前言
本篇文章来分析下 Kratos 中的 Mysql 接口封装，主要是对 [`go-sql-driver`](https://github.com/go-sql-driver/mysql) 此库的封装。在其中加入了 Metrics 统计、Tracing 及熔断机制嵌入的实现。思路非常的清晰，代码主要集中在 [`sql.go`](https://github.com/go-kratos/kratos/blob/master/pkg/database/sql/sql.go) 以及 [`mysql.go`](https://github.com/go-kratos/kratos/blob/master/pkg/database/sql/mysql.go) 中。

##	0x01	基础使用
`database/sql` 与 `go-sql-driver` 两者之间的关系，golang 的 `database/sql` 源码为数据库提供了一种抽象能力，然后让第三方库比如 `go-sql-driver` 去实现。

##  0x02	Metrics 统计
Metrics 的定义的结构 [代码在此](https://github.com/go-kratos/kratos/blob/master/pkg/database/sql/metrics.go)，包含了下面几个维度：
1.	`_metricReqDur`
2.	`_metricReqErr`
3.	`_metricConnTotal`
4.	`_metricConnCurrent`

```golang
var (
	_metricReqDur = metric.NewHistogramVec(&metric.HistogramVecOpts{
		Namespace: namespace,
		Subsystem: "requests",
		Name:      "duration_ms",
		Help:      "mysql client requests duration(ms).",
		Labels:    []string{"name", "addr", "command"},
		Buckets:   []float64{5, 10, 25, 50, 100, 250, 500, 1000, 2500},
	})
	_metricReqErr = metric.NewCounterVec(&metric.CounterVecOpts{
		Namespace: namespace,
		Subsystem: "requests",
		Name:      "error_total",
		Help:      "mysql client requests error count.",
		Labels:    []string{"name", "addr", "command", "error"},
	})
	_metricConnTotal = metric.NewCounterVec(&metric.CounterVecOpts{
		Namespace: namespace,
		Subsystem: "connections",
		Name:      "total",
		Help:      "mysql client connections total count.",
		Labels:    []string{"name", "addr", "state"},
	})
	_metricConnCurrent = metric.NewGaugeVec(&metric.GaugeVecOpts{
		Namespace: namespace,
		Subsystem: "connections",
		Name:      "current",
		Help:      "mysql client connections current.",
		Labels:    []string{"name", "addr", "state"},
	})
)
```

##	0x03	Tracing 的封装


##	0x04	熔断 Breaker 机制嵌入

##  0x05	参考


