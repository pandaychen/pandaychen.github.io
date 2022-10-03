---
layout: post
title: Golang 的分布式任务队列：Machinery （v1）分析（二）
subtitle: Machinery 应用场景梳理&&使用说明
date: 2021-05-01
author: pandaychen
catalog: true
tags:
    - Machinery
    - 队列
    - 异步队列
---

## 0x00  前言
通常，在设计业务模型中，Machinery 的 Worker 通常用于处理复杂 / 耗时的任务，即通过异步任务的方式来增加系统的吞吐量，减少同步请求的耗时。

## 0x01  应用场景 1：异步处理大量耗时任务
如业务场景中对涉及到大文件的用户数据导入及下载，数据删除等耗时场景，就比较适合用 Machinery 来做异步化处理：

-  用户数据导入：客户端上传数据集压缩包后，发送创建数据集请求，后台异步完成数据集导入逻辑
-  文件下载：后台拉取数据集并生成压缩包后，生成下载地址提供客户端下载
-  文件删除：客户端发起删除请求后，同步删除数据集记录，并异步清理数据集文件

以文件导入为例，流程如下：
-  用户客户端向 `Server` 模块发送文件导入请求，附带文件的中转区下载链接
-  `Server` 模块收到请求并处理
-  `Server` 模块发送文件导入异步任务，此时会触发 `Worker` 的异步处理流程：
   -  多个 `Worker` 模块实例分取 `1` 个导入文件任务，从中转区下载文件并导入数据集
   -  每个 `Worker` 实例执行导入文件任务
   -  `Worker` 的执行结果写入 `Backend`
-  `Server` 模块添加数据集记录
-  返回客户端请求处理成功，同步返回

![app-1](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/2022/machinery/machinery-app-1.png)

记得使用自定义的queueName来分配不同类型的数据操作任务，提高并发度

## 0x02  应用场景 2：海量数据求和

-  生成百万个整数
-  `Server` 模块发送整数累加异步任务，此时会触发 `Worker` 的异步处理流程
-  `Worker` 的执行结果写入 `Backend`

相关代码[见此](https://github.com/pandaychen/golang_in_action/blob/master/machinery/server.go)

从Redis可查询到运行结果：
```text
127.0.0.1:6379> get task_8bcf6fe3-5af4-4ded-8efa-3422d9a87754
"{\"TaskUUID\":\"task_8bcf6fe3-5af4-4ded-8efa-3422d9a87754\",\"TaskName\":\"sum\",\"State\":\"SUCCESS\",\"Results\":[{\"Type\":\"int64\",\"Value\":499999500000}],\"Error\":\"\",\"CreatedAt\":\"2022-10-03T06:47:53.178696786Z\",\"TTL\":0}"
```

## 0x03  一些重要的使用说明

1、支持的数据类型<br>
Machinery在将任务发送到Worker之前将其编码为JSON格式，任务执行结果也作为JSON编码的字符串存储在Backend。Machinery支持如下原生JSON类型:
```go
bool
int
int8
int16
int32
int64
uint
uint8
uint16
uint32
uint64
float32
float64
string
[]bool
[]int
[]int8
[]int16
[]int32
[]int64
[]uint
[]uint8
[]uint16
[]uint32
[]uint64
[]float32
[]float64
[]string
```


## 0x04  参考
