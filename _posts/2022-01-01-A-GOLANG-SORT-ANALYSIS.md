---
layout: post
title: 数据结构与算法回顾（七）：Golang 中的 Sort 算法实现解析
subtitle:   排序算法回顾
date: 2021-08-01
header-img: img/super-mario.jpg
author: pandaychen
catalog: true
tags:
  - 数据结构
  - Sort
---

## 0x00 前言
本文分析下golang的Sort算法实现

##  0x01    sort的典型使用
Golang 的 `sort.Sort` 使用了一个接口类型 `sort.Interface` 来指定通用的排序算法和可能被排序到的序列类型之间的约定。这个接口的实现由序列的具体表示和它希望排序的元素决定，通常排序的对象是Slice

####    case1：




##  0x02    分析
sort 包内部实现了四种基本的排序算法：
-   插入排序（insertionSort）
-   归并排序（symMerge）
-   堆排序（heapSort）
-   快速排序 （quickSort）


##  0x0 参考
-   [一文搞懂Go语言中的切片排序](https://tonybai.com/2020/11/26/slice-sort-in-go/)