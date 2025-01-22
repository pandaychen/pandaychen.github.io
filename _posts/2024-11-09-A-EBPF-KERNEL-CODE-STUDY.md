---
layout:     post
title:      EBPF 内核态代码学习（一）：
subtitle:
date:       2024-11-09
author:     pandaychen
header-img:
catalog: true
tags:
    - eBPF
---

##  0x00    前言
linux内核代码提供了`runqslower`的工具，该工具用于展示在CPU run队列中停留的时间大于某一值的任务（哪些进程的调度延迟超过了特定的阈值），有两个版本：

-   [bcc基于libbpf的版本](https://github.com/iovisor/bcc/blob/master/libbpf-tools/runqslower.c)
-   [内核实现的版本](https://github.com/torvalds/linux/blob/master/tools/bpf/runqslower/runqslower.bpf.c)：有若干新特性，比如`v5.11`版本的提供的helper方法：[`bpf_task_storage_get`](https://docs.ebpf.io/linux/helper-function/bpf_task_storage_get/)

主要涉及到如下hook点：
-   `tp_btf/sched_wakeup`
-   `tp_btf/sched_switch`
-   `tp_btf/sched_wakeup_new`

####    runslower工具

1、`runqslower 10000`：检测哪些进程的run delay超过`10ms`

```BASH
[root@VM-X-X-tencentos libbpf-tools]# ./runqslower 10000
Tracing run queue latency higher than 10000 us
TIME     COMM             TID           LAT(us)
17:39:01 process1          262259          27431
17:39:01 process2          262260          27366
17:39:01 process3          262262          27255
```

2、`./runqslower 10000 -P`：增加`-P`选项，可以知道当前进程的调度延迟是由前面执行的哪个任务导致的

```BASH
[root@VM-X-X-tencentos libbpf-tools]# ./runqslower 10000 -P
Tracing run queue latency higher than 10000 us
TIME     COMM             TID           LAT(us) PREV COMM        PREV TID
18:40:23 heartbeat        1974944          60934 swapper/1        0     
18:40:24 heartbeat        1974944          79923 swapper/1        0     
18:40:24 cpulimit         1775688          35520 swapper/4        0     
18:40:39 awk              1975065          79925 swapper/7        0     
18:40:39 grep             1975064          79925 swapper/2        0     
18:40:39 ps               1975061          74938 swapper/3        0     
18:40:39 tmanager-servic  1491             53124 swapper/1        0     
18:40:39 tagentV1.0       1775693          73167 swapper/5        0     
18:40:40 readlink         1975066          84938 swapper/3        0     
18:40:40 ps               1975068          90918 swapper/1        0     
18:40:40 tmanager-servic  1491            49719 runqslower       1974929
```

大致原理如下，在`runqslower`发现调度延迟的情况下，必然是由于其他的task的抢占CPU导致。那么可能是某一个task或者某几个task，那么在抓到了调度延迟的时候，把前面的task也dump出来，那么大概率是可以发现是哪个task抢占的CPU时间，这样就可以发现是由于哪个进程的影响导致了延迟

##  0x01    基于bcc的版本


##  0x0  参考
-   [透过Tracepoint理解内核 - 调度器框架和性能](https://zhuanlan.zhihu.com/p/143320517)
-   [runqslower实现：kernel](https://github.com/torvalds/linux/blob/master/tools/bpf/runqslower/runqslower.bpf.c)
-   [runqslower实现：bcc](https://github.com/iovisor/bcc/blob/master/libbpf-tools/runqslower.bpf.c)
-   [Helper function bpf_task_storage_get](https://docs.ebpf.io/linux/helper-function/bpf_task_storage_get/)