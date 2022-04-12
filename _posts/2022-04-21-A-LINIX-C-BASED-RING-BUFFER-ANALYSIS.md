---
layout: post
title: 数据结构与算法回顾（四）：环形内存缓冲区 ringbuffer（TODO FIX）
subtitle: 一种高效进程间通信的机制：环形内存缓冲区
date: 2022-04-05
header-img: img/super-mario.jpg
author: pandaychen
catalog: true
tags:
  - 环形队列
  - Ring Buffer
  - Circular Buffer
---

## 0x00 前言
环形队列是一种 FIFO 数据结构，适合内存 / 共享内存的存储场景，是项目中解耦模块间（进程间）通信的可用手段之一。通常也称为 Ring Buffer、Circular Buffer。下图描绘了一个 A 24-byte keyboard circular buffer：

![image](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/2022/queue/Circular_Buffer_Animation.gif)

## 0x01 实现
ringbuffer 的实现主要依赖于读写指针的移动（head-ReadIndex/tail-WriteIndex）：
- 初始化一块定长的内存（或共享内存）作为存储空间，长度即为 `m_size`
- 通过 `mod m_size` 得出 ReadIndex/WriteIndex 的相对位置，进而实现 "环形" 的机制
- 写入 / 读取操作时需要考虑边界情况，写入需要移动 `tail` 指针，读取需要移动 `head` 指针

![image](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/blob/master/blog_img/2022/queue/ringbuffer-2.png)

##  0x02  现网应用
基于 ringbuffer 可以实现无锁的通信，在现网项目中，会遇到进程间通信的场景，及两个进程（进程 `A` 和进程 `B`）需要进行双向数据通信，如何无锁化实现呢？这就可以借用 ringbuffer，实现思路如下：
1.  创建两块固定大小的共享内存（共享内存 `X` 和共享内存 `Y`），每个共享存储单元的数据头信息结构为 `DataHead`，数据体信息为 `DataUnit`
2.  当进程 `A` 想传送数据体信息给进程 `B` 时，进程 `A` 向共享内存 `X` 写入数据体信息，变更共享内存 A 中的 `DataHead` 中的 `tail` 信息，进程 `B` 从共享内存 `X` 读出数据体信息，变更共享内存 `X` 中的 `DataHead` 中的 `head` 信息
3.  同样的方式，当进程 `B` 想传送数据体信息给进程 `A` 时，进程 `B` 向共享内存 `Y` 写入数据体信息，变更共享内存 `Y` 中的 `DataHead` 中的 `tail` 信息，进程 `A` 从共享内存 `Y` 读出数据体信息，变更共享内存 `Y` 中的 `DataHead` 中的 `head` 信息
4.  整个交互过程中，进程 `A` 与进程 `B` 无缝的通过 ringbuffer 进行通信，实现了进程间通信效率的最大化


流程图如下：
![image](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/2022/queue/ring-buffer-shm-2-process.png)

##  0x03  代码实现
整个存储结构如下：
![ring-buffer-3](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/2022/queue/ringbuffer-3.png)

####  数据结构定义
每个存储在内存中的数据都包括如下两个结构 `DataHead` 和 `DataUnit`：
- `DataHead`：内存数据头结构
- `DataUnit`：内存数据体结构（变长）

```cpp
struct NodeDataHead
{
	int iSize;
	int iTail;    // 写入数据更新位置
	int iHead;  // 读取数据更新位置
	int iOffset;
};

struct NodeDataUnit
{
	int iLen;   // 可变长
  	char *pData;
};
```

##  0x04  总结

##  0x05  参考
- [Circular buffer](https://en.wikipedia.org/wiki/Circular_buffer)