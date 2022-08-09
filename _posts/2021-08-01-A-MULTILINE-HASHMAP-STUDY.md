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
  - hashtable
---

## 0x00 前言
hashtable 是一种非常高效的数据结构，它通过散列函数将 key 映射成数组下标以达到 `O(1)` 算法复杂度的高效数据结构，通常对于不同的 key 哈希函数算出的结果出现哈希碰撞，解决方式通常有：开放地址法、再哈希法、链地址法。多阶（级）Hashtable 是一种极其精妙的实现，常用于海量数据（如 UIN、即时通讯关系链等）的存储，此外，结合 Linux 共享内存机制，可以方便的实现在游戏后台中保存用户状态（进程崩溃恢复）。多阶（级）hash 有几个特点：

1.  容量固定：开发者需要预估好自己系统的存储规模
2.  利用率极高：按照公司同事的测试，`50` 阶多级 hash 的利用率至少可以达到 `95%` 了
3.  可根据开发场景灵活设计，可选择内存或者是共享内存来存储
4.  根据 KeyValue 是否变长，又引申出几种变体；
  - 固定 Size，多阶 hash 每个节点包含了 value
  - 不固定 Size，单独开辟一大块数据区域，按照 KeyValue 分离的方式存储，多阶 hash 的 Key 节点包含指向数据区域的 Value 的指针

## 0x01 多阶 Hash 简介

多阶 hash 实际上是一个（相对平滑）锯齿状数组。从第一层级开始执行插入时，将某一层冲突的 key，交由下一层进行填充，直到最终层。如下所示：

![image](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/2021/hashtable/ml-hashtable-2.png)


多阶 Hash 的特点是：

- 每一行是一阶，每一行的元素个数都是素数个（连续素数，需要取模均匀）
- 数组的每个节点用于存储数据的内容，其中，节点存储 `int` 类型的 key/hash_code 的索引以及数据，通常适合定长数据的场景
- 也可以选择索引与数据分离存储，适合不定长数据的场景
- 可以使用内存，也可以使用共享内存机制实现（不使用指针，使用一大块固定 size 的共享内存）
- 使用率高，实测 `20` 阶的结构使用率可以达到 `95%`
- 查找和插入的复杂度都是 `O(n)`，其中 `n` 是阶数

#### 插入与冲突解决
多阶 hash 的插入方式为：依次从第 `1` 阶开始往后查找是否已存在当前阶中，如果找到则直接覆盖更新（或者返回失败，依赖业务逻辑）数据，如果查找完所有的阶都没有找到记录，则找出从第 `1` 阶到最后一阶可以分配给该 key 并且没有被使用的位置插入该记录。通常情况下会上面的阶数先被填满，然后再逐步填下面的阶数

#### 查找

1.  先将 key 在第 `1` 阶内取模，看是否是这个元素，如果这个位置为空，直接返回不存在；如果是这个 key，则返回这个位置，查找结束
2.  如果这个位置有元素，但是已存在的 Key 与带查找的 key 不同，则说明 hash 冲突，再到第二阶去找
3.  循环往复，直到查找完最后一阶，任未查找到，退出

即先从第一阶开始依次往后查找，如果找到就返回，没找到就到下一阶继续查找，直到查找完所有的哈希桶返回查找失败。


![insert](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/2021/hashtable/ml-hashtable-1.png)

## 0x02 素数集中原理算法

对于多阶 Hash 而言，每一阶究竟应该选择多少个元素？答案是素数集中原理（`6N-1/6N+1` 素数选择）。例如，假设每阶最多 `1000` 个元素，一共 `10` 阶，则算法选择十个比 `1000` 小的最大素数，从大到小排列，以此作为各阶的元素个数。通过素数集中的算法得到的 `10` 个素数分别是：`997`、`991`、`983`、`977`、`971`、`967`、`953`、`947`、`941` 和 `937`。从结果可见，虽然是锯齿数组，各层之间的数量差别并不是很多。

##  0x03  源码分析

##  0x04  源码分析

##  0x05  一些小技巧

####  减少 hash 算法的冲突率
针对于不同的字符串，可能 hash 出同一个值，如何减小这种冲突率？比如 time33 算法，`100W` 数量级的冲突率约为 `0.00015`，从测试结果来看，`murmur_hash` 是相对较好的 [选择](https://zh.wikipedia.org/zh-hans/Murmur 哈希)，该算法支持设置初始值可以生成多个不同的 hash 值，我们采用类似 [Cuckoo hashing](https://en.wikipedia.org/wiki/Cuckoo_hashing) 的思路：

1、对一个字符串 `str1`, 使用 `murmur_hash` 同时 hash 出 `3` 个值（足够了），记为 `h1`、`h2` 和 `h3`<br>
```cpp
h1 = murmur_hash(str, 0);
h2 = murmur_hash(str, 1);
h3 = murmur_hash(str, 2);
```

2、`h1` 用来作多阶 hashtable 中 key 的定位，计算 hash 要保存的位置。而 `h2` 和 `h3` 仅用于辅助比较。对于一个 key, 需要满足 `3` 个 hash 值都一致时才认为是需要找的 key。如此做法，可以将冲突率降至约 `2.24e-12`，基本上满足我们的优化需求了 <br>

##  0x06  总结
本文讨论了多阶 hash 的实现思路，这里总结下：

####  优点
- 查找和插入时间稳定高效，正比于阶数虽然不是最高效的，解决了链地址法冲突太多退化成链表操作效率不稳定
- 实现简单，使用多维数组实现
- 便于扩容，只要在加一阶数组就可以完成扩容

####  缺点
- 容量有限，虽然可以很简便的扩容，但是数量总是有上限的
- 扩展性一般，在每一阶都填满的情况下会出现 key 无法存储，相比于链地址法碰撞的情况可以增加到链表末尾相比扩展性更差
- 由于数组是定长的，所以定义元素的时候需要以最大的 value 来定义元素大小（如果不采用索引与数据分离的方式来实现的话）


## 0x07 参考

- [介绍一种服务器缓存结构 --- 多级 Hash](https://software.intel.com/content/www/cn/zh/develop/articles/introducing-server-cache-structure-multilevel-hash.html)
- [mem_hash](https://github.com/zfengzhen/mem_hash)
- [复习：多阶 hash 表](http://ahfuzhang.blogspot.com/2012/09/hash.html)
- [使用共享内存的多级哈希表的一种实现](http://www.cppblog.com/lmlf001/archive/2007/09/08/31858.html)
- [多阶 hash 实现与分析](http://www.xiaocc.xyz/2020-07-20/%E5%A4%9A%E9%98%B6hash%E5%AE%9E%E7%8E%B0%E5%88%86%E6%9E%90/)
- [Math - 数学基础知识素数 Prime](https://houbb.github.io/2017/08/23/math-02-common-prime-02)
