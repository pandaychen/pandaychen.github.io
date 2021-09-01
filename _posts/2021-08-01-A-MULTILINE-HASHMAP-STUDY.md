---
layout: post
title: 数据结构与算法回顾（一）：多阶 hash
subtitle: 一种高利用率的 hash 存储结构分析
date: 2021-08-01
header-img: img/super-mario.jpg
author: pandaychen
catalog: true
tags:
  - 数据结构
  - hashmap
---

## 0x00 前言

多阶（级）Hash 是我司使用最普遍的数据结构，常用于海量数据（如 UIN、关系链等）的存储，此外，结合 Linux 共享内存机制，可以方便的实现在游戏后台中保存用户状态（进程崩溃恢复）。

## 0x01 多阶 Hash 简介

多阶 hash 实际上是一个（相对平滑）锯齿状数组，如下所示：

```JavaScript
■■■■■■■■■■■■■■■
■■■■■■■■■■■■■
■■■■■■■■■■■■
■■■■■■■■■■■
```

多阶 Hash 的特点是：

- 每一行是一阶，每一行的元素个数都是素数个（连续素数，需要取模均匀）
- 数组的每个节点用于存储数据的内容，其中，节点存储 `int` 类型的 key/hash_code 的索引以及数据，通常适合定长数据的场景
- 也可以选择索引与数据分离存储，适合不定长数据的场景
- 创建多阶 Hash 的时候，用户通过参数来指定有多少阶，每一阶最多多少个元素

#### 插入与冲突解决

#### 查找

1.  先将 key 在第一阶内取模，看是否是这个元素，如果这个位置为空，直接返回不存在；如果是这个 key，则返回这个位置，查找结束
2.  如果这个位置有元素，但是已存在的 Key 与带查找的 key 不同，则说明 hash 冲突，再到第二阶去找
3.  循环往复，直到查找完最后一阶，任未查找到，退出

## 0x02 素数集中原理算法

对于多阶 Hash 而言，每一阶究竟应该选择多少个元素？答案是素数集中原理（`6N-1/6N+1` 素数选择）。例如，假设每阶最多 `1000` 个元素，一共 `10` 阶，则算法选择十个比 `1000` 小的最大素数，从大到小排列，以此作为各阶的元素个数。通过素数集中的算法得到的 `10` 个素数分别是：`997`、`991`、`983`、`977`、`971`、`967`、`953`、`947`、`941` 和 `937`。从结果可见，虽然是锯齿数组，各层之间的数量差别并不是很多。

## 0x03 参考

- [介绍一种服务器缓存结构 --- 多级 Hash](https://software.intel.com/content/www/cn/zh/develop/articles/introducing-server-cache-structure-multilevel-hash.html)
- [mem_hash](https://github.com/zfengzhen/mem_hash)
- [复习：多阶 hash 表](http://ahfuzhang.blogspot.com/2012/09/hash.html)
- [使用共享内存的多级哈希表的一种实现](http://www.cppblog.com/lmlf001/archive/2007/09/08/31858.html)
- [多阶 hash 实现与分析](http://www.xiaocc.xyz/2020-07-20/%E5%A4%9A%E9%98%B6hash%E5%AE%9E%E7%8E%B0%E5%88%86%E6%9E%90/)
