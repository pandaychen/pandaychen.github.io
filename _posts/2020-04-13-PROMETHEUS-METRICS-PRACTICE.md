---
layout: post
title: 理解 Prometheus 的基本数据类型及应用（实践篇）
subtitle: Prometheus 及 promql 的一些实践
date: 2020-04-13
author: pandaychen
header-img: img/golang-horse-fly.png
catalog: true
category: false
tags:
  - Prometheus
  - Metrics
---

## 0x00 前言
本文是对前一篇文章[]()的实践以及深入理解，推荐如下几篇文章：

- [PromQL 简明教程](https://www.kawabangga.com/posts/4408)

##  0x01  再看四类指标



##  0x02  promql的典型应用：引入



##  0x03  promql的典型应用：实践

##  0x04  

六、PromQL要先 rate() 再 sum()，不能 sum() 完再 rate()
这背后与 rate() 的实现方式有关，rate() 在设计上假定对应的指标是一个 Counter，也就是只有 incr(增加) 和 reset(归0) 两种行为。而做了 sum() 或其他聚合之后，得到的就不再是一个 Counter 了，举个例子，比如 sum() 的计算对象中有一个归0了，那整体的和会下降，而不是归零，这会影响 rate() 中判断 reset(归0) 的逻辑，从而导致错误的结果。写 PromQL 时这个坑容易避免，但碰到 Recording Rule 就不那么容易了，因为不去看配置的话大家也想不到 new_metric 是怎么来的。

Recording Rule规则：一步到位，直接算出需要的值，避免算出一个中间结果再拿去做聚合。



## 0x07 参考

- [一文带你了解 Prometheus](https://cloud.tencent.com/developer/article/1999843)
- [一文搞懂 Prometheus 的直方图](https://juejin.im/post/5d492d1d5188251dff55b0b5)
- [如何区分 prometheus 中 Histogram 和 Summary 类型的 metrics？](https://www.cnblogs.com/aguncn/p/9920545.html)
- [Metrics 设计：teleport](https://goteleport.com/teleport/docs/metrics-logs-reference/)
- [Lock-free Observations for Prometheus Histograms](https://grafana.com/blog/2020/01/08/lock-free-observations-for-prometheus-histograms/)
- [golang API](https://godoc.org/github.com/prometheus/client_golang/prometheus)
-  [HISTOGRAMS AND SUMMARIES](https://prometheus.io/docs/practices/histograms/)
-  [prometheus 的内置函数](https://prometheus.io/docs/prometheus/latest/querying/functions/)
- [Prometheus 的九个坑](https://www.jianshu.com/p/c99375ef40aa)
- [PromQL 简明教程](https://www.kawabangga.com/posts/4408)

转载请注明出处，本文采用 [CC4.0](http://creativecommons.org/licenses/by-nc-nd/4.0/) 协议授权
