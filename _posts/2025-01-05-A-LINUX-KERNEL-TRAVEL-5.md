---
layout:     post
title:  Linux 内核之旅（五）：内核的可观测技术
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

####    性能追踪
-   宏观：通过全链路监控找出整个分布式系统中的瓶颈组件
-   微观：快速地找出进程内的瓶颈函数，从（内核）代码层面直接寻找调用次数最频繁、耗时最长的函数，通常它就是性能瓶颈

####    linux tracing技术
![linux-tracing-arch](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/trace/linux_trace_arch.png)

1、观测数据源，分为指标&事件两类

指标观测

![metrics](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/trace/linux_metric.png)

事件观测

![trace](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/trace/linux_trace.png)

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

####    ftrace实战
比如，使用ftrace追踪X86系统下，`open`/`openat`内核函数调用链：

```BASH
# 挂载 tracefs（若未自动挂载）
sudo mount -t tracefs nodev /sys/kernel/tracing
# 进入跟踪目录
cd /sys/kernel/tracing

#启用函数调用图跟踪器
echo function_graph > current_tracer   

#设置目标追踪函数
echo __x64_sys_openat > set_graph_function

#注意：若追踪函数不存在会设置报错
#echo do_sys_openat > set_graph_function 
#-bash: echo: write error: Invalid argument

cat set_graph_function
__x64_sys_openat

# 过滤特定进程，只获取目标进程 PID（例如跟踪 PID 为 311588 的进程）
echo 311588 > set_ftrace_pid

cat set_ftrace_pid 
311588

#设置跟踪缓冲区大小（避免溢出），调整缓冲区为 16MB
echo 16384 > buffer_size_kb

# 启动跟踪，清空旧数据
echo > trace

# 开启跟踪
echo 1 > tracing_on

# 关闭跟踪
echo 0 > tracing_on

# 输出最终结果，调用路径
cat trace
```

最终得到`openat`系统调用链如下所示：

```TEXT
[root@VM-X-X-tencentos tracing]# cat trace|grep open
 7)  write-311588  |               |  __x64_sys_openat() {
 7)  write-311588  |               |    do_sys_openat2() {
 7)  write-311588  |               |      do_filp_open() {
 7)  write-311588  |               |        path_openat() {
 7)  write-311588  |               |          open_last_lookups() {
 7)  write-311588  |               |            lookup_open.isra.0() {
 7)  write-311588  |               |          do_open() {
 7)  write-311588  |               |            may_open() {
 7)  write-311588  |               |            vfs_open() {
 7)  write-311588  |               |              do_dentry_open() {
 7)  write-311588  |               |                security_file_open() {
 7)  write-311588  |               |                  selinux_file_open() {
 7)  write-311588  |               |                  bpf_lsm_file_open() {
 7)  write-311588  |               |                ext4_file_open() {
 7)  write-311588  |               |                  dquot_file_open() {
 7)  write-311588  |   0.254 us    |                    generic_file_open();
```

##  0x02    bpftrace


##  0x03    perf
![perf-arch]()

####    perf工作过程
perf采样过程大概分为两步：

1.  调用 `perf_event_open` 来打开一个 event 文件，主要工作由[`perf_event_open`](https://elixir.bootlin.com/linux/v4.11.6/source/tools/testing/selftests/powerpc/pmu/event.c#L16)完成，包括创建各种event内核对象、创建各种event文件句柄以及指定采样处理回调
2.  调用 `read`、`mmap`等系统调用读取内核采样回来的数据

![perf_event](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/trace/perf_work.jpg)

当 `perf_event_open` 创建事件对象并打开后，硬件上发生的事件就可以触发执行了。`perf_event_open`给CPU指定了中断处理函数，这个内核注册相应的硬件中断处理函数是 `perf_event_nmi_handler`。这样 CPU 硬件会根据 `perf_event_open` 调用时指定的周期发起中断，调用 `perf_event_nmi_handler` 通知内核进行采样处理。具体过程是访问该进程的IP寄存器的值（也就是下一条指令的地址），通过分析该进程的可执行文件，可以得知每次采样的IP值处于哪个函数的内部，最后内核和硬件一起协同合作，定时将当前正在执行的函数，以及函数完整的调用链路都给记录下来

####    perf实战

1、perf stat（计数模式）

```BASH
#通过 -e 指定 Kernel Tracepoint Events，perf stat 可以统计程序执行的系统调用
[root@VM-X-X-centos ~]# perf stat -e 'sched:sched_process_*' -a sleep 5

 Performance counter stats for 'system wide':

                22      sched:sched_process_free                                    
                28      sched:sched_process_exit                                    
                72      sched:sched_process_wait                                    
                90      sched:sched_process_fork                                    
                56      sched:sched_process_exec                                    
                 0      sched:sched_process_hang                                    

       5.003716058 seconds time elapsed
```

##  0x04    总结


##  0x05  参考
-   [七张图看懂 Linux profiling 机制](https://tinylab.org/linux-profiling-methods-overview)
-   [从Ftrace开始内核探索之旅](https://github.com/mz1999/blog/blob/master/docs/ftrace.md)
-   [问题排查利器：Linux 原生跟踪工具 Ftrace 必知必会](https://www.ebpf.top/post/ftrace_tools/)
-   [【一文秒懂】Ftrace系统调试工具使用终极指南](https://www.cnblogs.com/-Donge/p/17981595)
-   [ftrace基本用法](https://tinylab.org/ftrace-usage/)
-   [Linux可观测性](https://qiankunli.github.io/2019/11/25/linux_observability.html#tracepoint-%E5%92%8C-kprobe)
-   [一文学会ftrace的基础用法](https://www.daodaodao123.com/?p=959)
-   [2.4 perf 的使用](https://hotttao.github.io/posts/linux/linux_perf/06_perf_use/)
-   [CPU平均负载为多少更合理？](https://mp.weixin.qq.com/s/utbtKusx-gBgemh94f6trg)