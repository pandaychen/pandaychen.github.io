---
layout: post
title: prometheus 查询 && 应用实践
subtitle: 如何更好的使用 prometheus
date: 2022-05-01
header-img: img/super-mario.jpg
author: pandaychen
catalog: true
tags:
  - prometheus
---

##  0x00    前言
本文梳理下 prometheus 查询及指标应用的最佳实践。

##  0x01    相关概念回顾

####    时序数据库

####    采样样本
Prometheus 会定期去对数据进行采集，每一次采集的结果都是一次采样样本（sample），这些数据会被存储为时间序列，也就是带有时间戳的 value stream，这些 value stream 归属于自己的监控指标。采集样本包括了 `3` 部分：
-   监控指标（metric）
-   毫秒时间戳（timestamp）
-   样本值（value）

####    关于监控指标的解读
监控指标被表示为下面的格式：

```text
metric_name {label_name_1=label_value_1, label_name_2=label_value_2, ...}
```

解释：
-   `metric_name`：指明监控的内容
-   `label_value_xxx`：用于声明这个监控内容中不同维度的值

使用常见的二维坐标系（xxx 坐标系）举例，有 `X`，`Y` 两个轴，上面有两点 `A` 和 `B`，坐标分别为 `(1, 3)` 和 `(2, 1)`：

```text
Y
  ^
  │   . A (1, 3)
  │
  │     . B (2, 1)
  v
    <-----------------> X
```


对应于 Prometheus，`metric_name` 就是xxx 坐标系，而`label_name_1` 即为 `X`，`label_name_2` 为 `Y`。需要注意的是 `A `和 `B` 两个点并不代表采样点，而是**监控指标**。

**可以想象在此坐标系中还存在一条虚拟的时间轴，分别从 `A/B` 两点从屏幕外垂直屏幕进去，在这两条虚拟的时间轴上，每一个点就是一个采样点，采样点上会带一个毫秒时间戳和一个值，这个值就是样本的值**

对于 Prometheus 而言，这里存在两个时间序列，分别为：

-   坐标系 `{"X"="1","Y"="3"}`
-   坐标系 `{"X"="2","Y"="1"}`

在 Prometheus 中，样本的值必须为 `float64` 类型；此外，建议标签值不要使用一个数量非常多的值，否则会造成时间序列数量的极度膨胀。标签的值应该越简单越好

##  0x0 参考
-   [彻底理解 Prometheus 查询语法](https://blog.csdn.net/zhouwenjun0820/article/details/105823389)
-   [Prometheus 操作指南](https://github.com/yunlzheng/prometheus-book)
-   [理解时间序列](https://github.com/yunlzheng/prometheus-book/blob/master/promql/what-is-prometheus-metrics-and-labels.md)
-   [Prometheus Metrics 设计的最佳实践和应用实例，看这篇够了](https://cloud.tencent.com/developer/article/1639138)