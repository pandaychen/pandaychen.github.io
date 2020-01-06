---
layout:     post
title:      Python性能优化
subtitle:   使用heapy统计堆上的对象（内存）
date:       2019-10-20
author:     pandaychen
catalog:    true
description:	使用heapy统计堆上的对象，如何避免dict内存扩容导致的OOM
tags:
    - 性能优化
    - Python
---

##  背景
最近工作中遇到一个场景，使用`python`写的脚本做`kafka`的消费端，使用的是多进程模式，父进程从`kafka`收，通过`multiprocessing.Queue`发给子进程处理，处理的数据是基于会话的，子进程用一个`dict`来存储。KEY为会话ID，VALUE为一个list（模拟`queue`的方式实现了一个字符数组队列），根据指定的分隔符做`push`和`pop`操作。但是随着会话数量的增加，`dict`使用的内存量也是无限扩展，根据先前遇到的case来看，这样无限增长下去的后果就是被系统给OOM。于是，这就引入了一个问题，如何监控`Heap`的使用量？如何避免`dict`内存的无限扩张？


##  监控篇
[Python高性能编程](https://book.douban.com/subject/27064848/)这本书中介绍过一个模块，`Heapy`。很适合拿来做`Heap`内存用量统计。<br>
`guppy` 获取内存使用的各种对象占用情况<br>
`guppy` 可以打印各种对象所占空间大小，如果`python`进程中有未释放的对象，造成内存占用升高，可通过`guppy`查看。<br>

看下面的例子：
``` python
#coding=utf8

from guppy import hpy

def calc_pure_python():
    hp=hpy()
    h = hp.heap()
    print h
    print

    d={}
    for i in range(1,2000000):
        d[i]=i	
    h = hp.heap()
    print h
    print

if __name__=='__main__':
	calc_pure_python()

```

<br><br>
运行下输入结果，可以看到最终200w个数据，大约占用了151987464/1024/1024=144M内存。
``` bash
Partition of a set of 25949 objects. Total size = 3325000 bytes.
 Index  Count   %     Size   % Cumulative  % Kind (class / dict of class)
     0  11775  45   933896  28    933896  28 str
     1   5841  23   470280  14   1404176  42 tuple
     2    324   1   277728   8   1681904  51 dict (no owner)
     3     67   0   213064   6   1894968  57 dict of module
     4    199   1   210856   6   2105824  63 dict of type
     5   1630   6   208640   6   2314464  70 types.CodeType
     6   1595   6   191400   6   2505864  75 function
     7    199   1   177008   5   2682872  81 type
     8    124   0   135328   4   2818200  85 dict of class
     9   1045   4    83600   3   2901800  87 __builtin__.wrapper_descriptor
<91 more rows. Type e.g. '_.more' to view.>

Partition of a set of 2025820 objects. Total size = 151987464 bytes.
 Index  Count   %     Size   % Cumulative  % Kind (class / dict of class)
     0    331   0 100942984  66 100942984  66 dict (no owner)
     1 2000138  99 48003312  32 148946296  98 int
     2  11777   1   934024   1 149880320  99 str
     3   5840   0   470216   0 150350536  99 tuple
     4     67   0   213064   0 150563600  99 dict of module
     5    199   0   210856   0 150774456  99 dict of type
     6   1630   0   208640   0 150983096  99 types.CodeType
     7   1594   0   191280   0 151174376  99 function
     8    199   0   177008   0 151351384 100 type
     9    124   0   135328   0 151486712 100 dict of class
<91 more rows. Type e.g. '_.more' to view.>
```

##  优化篇
为了避免`dict`的无限内存扩张，我的优化主要从下面几个方面着手的：
1.  预估好单机的可用内存量可以承载多大规模的会话，设置处理的会话数量上限，在进程中做统计，当会话量达到上限的80%时，触发告警机制；
2.  对`Kafka`的消费端做分组，生产端对会话ID做`Topic`绑定，确定一个唯一的会话只发到一个固定的Partition上，这就保证了一个会话数据，只被一个固定的消费端进程处理；
3.  对`dict`的本身使用上，定时器或者LRU的处理是必不可少的，实现上可以用采用最小堆或TimeWheel，定期清理超时的Key。


##  后记

转载请注明出处，本文采用 [CC4.0](http://creativecommons.org/licenses/by-nc-nd/4.0/) 协议授权