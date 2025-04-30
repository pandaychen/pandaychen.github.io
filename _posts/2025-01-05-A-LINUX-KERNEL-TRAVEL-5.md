---
layout:     post
title:  Linux 内核之旅（五）：内核的可观测
subtitle:   内核追踪的工具入门：ftrace/bpftrace/perf
date:       2025-01-05
author:     pandaychen
header-img:
catalog: true
tags:
    - Linux
---

##  0x00    前言
追踪类调试工具鸟瞰图

![trace-tool](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/trace/tracing.jpg)

##  0x01    ftrace

####    工作原理
Ftrace的框架图如下，ftrace包括多种类型的tracers，每个tracer完成不同的功能，将这些不同类型的tracers注册进入ftrace framework，各类tracers收集不同的信息，并放入到Ring buffer缓冲区以供调用

![ftrace](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/trace/ftrace-overview.png)

从上图可知，Ftrace是基于`debugfs`调试文件系统的，需要先挂载`debugfs`： `mount -t debugfs none /sys/kernel/debug`

```BASH
[root@VM-163-30-centos ]# ls /sys/kernel/debug/
acpi  boot_params  dynamic_debug  fnic  kprobes  rssd            suspend_stats  usb             x86
bdi   dma_buf      extfrag        hid   mce      sched_features  tracing        wakeup_sources
```

####    ftrace的hook机制
1、静态插桩方式

在Kernel中打开了`CONFIG_FUNCTION_TRACER`功能后，会增加`-pg`编译选项，编译器会在每个内核函数的入口处调用一个特殊的汇编函数`mcount` 或 `__fentry__`，如果跟踪功能被打开，`mcount/fentry` 会调用当前设置的 tracer，tracer将不同的数据写入ring buffer

2、动态插桩方式

启用了`CONFIG_DYNAMIC_FTRACE`选项，编译内核时所有的`mcount/fentry`调用点都会被收集记录。编译时记录所有被添加跳转指令的函数，这里表示所有支持追踪的函数。在内核的初始化启动过程中，会根据编译期记录的列表，将`mcount/fentry`调用点替换为`NOP`指令（ no-operation）并直接转到下一条指令。因此在没有开启跟踪功能的情况下，Ftrace不会对内核性能产生任何影响。在开启追踪功能时，Ftrace才会将`NOP`指令动态替换为`mcount/fentry`（这里的动态是指的动态修改函数指令），以实现追踪功能

![dynamic](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/trace/ftrace.png)

##  0x02    bpftrace


##  0x03    perf

##  0x04  参考
-   [七张图看懂 Linux profiling 机制](https://tinylab.org/linux-profiling-methods-overview)
-   [从Ftrace开始内核探索之旅](https://github.com/mz1999/blog/blob/master/docs/ftrace.md)
-   [问题排查利器：Linux 原生跟踪工具 Ftrace 必知必会](https://www.ebpf.top/post/ftrace_tools/)
-   [【一文秒懂】Ftrace系统调试工具使用终极指南](https://www.cnblogs.com/-Donge/p/17981595)
-   [ftrace基本用法](https://tinylab.org/ftrace-usage/)
-   [Linux可观测性](https://qiankunli.github.io/2019/11/25/linux_observability.html#tracepoint-%E5%92%8C-kprobe)
-   [一文学会ftrace的基础用法](https://www.daodaodao123.com/?p=959)