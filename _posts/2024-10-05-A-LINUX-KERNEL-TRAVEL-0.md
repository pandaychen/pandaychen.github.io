---
layout:     post
title:  Linux 内核之旅：基础知识
subtitle:
date:       2024-10-05
author:     pandaychen
header-img:
catalog: true
tags:
    - Linux
---

##  0x00    前言

##  0x01    双向链表

```CPP
struct list_head {
	struct list_head *next, *prev;
};
```

##  0x0  参考
-   [Linux 内核中的数据结构](https://blog.csdn.net/Rong_Toa/article/details/115110811)
-   [内核基础设施——hlist_head/hlist_node结构解析](https://linux.laoqinren.net/kernel/hlist/)