---
layout:     post
title:  Linux 内核之旅（七）：虚拟内存管理（下）
subtitle:   内核视角的虚拟内存管理
date:       2025-03-01
author:     pandaychen
header-img:
catalog: true
tags:
    - Linux
    - Kernel
---

##  0x00    前言
前文[Linux 内核之旅（三）：虚拟内存管理（上）](https://pandaychen.github.io/2024/11/05/A-LINUX-KERNEL-TRAVEL-2/)学习了进程虚拟内存空间在内核中的布局以及管理，本文继续学习下内核态的虚拟内存空间的布局及管理

对于进程虚拟内存空间而言，不同进程之间的虚拟内存空间是相互隔离的，彼此之间相互独立（相互无感知），使得进程以为自己拥有所有的内存资源。而内核态虚拟内存空间是所有进程共享的，不同进程进入内核态之后看到的虚拟内存空间全部是一样的

##  0x01  内核态虚拟空间地址布局  
由于内核会涉及到物理内存的管理，**有一个错误结论是只要进入了内核态就开始使用物理地址了**，进程进入内核态之后使用的仍然是虚拟内存地址，只不过在内核中使用的虚拟内存地址被限制在了内核态虚拟内存空间范围中

##  0x0		符号表相关  
内核提供了符号与地址的映射关系（内核只使用地址，符号表便于阅读&&调试），如DNS域名系统，通常，内核提供了有两类符号（映射）表：

-	`/boot/System.map-$(uname -r)`：包含整个内核镜像的符号表，是磁盘上真实文件
-	`/proc/kallsyms`：不仅包含内核镜像符号表，还包含所有动态加载模块的符号表（若函数被编译器内联inline或优化掉，则可能不存在于`/proc/kallsyms`），读取时内核动态生成

```cpp
[root@VM-X-X-centos ~]# ll -rth /boot/System.map-$(uname -r)
-rw-r--r-- 1 root root 4.6M Nov 26  2021 /boot/System.map-5.4.119-1-tlinux4-0008
[root@VM-X-X-centos ~]# ll -rth /proc/kallsyms
-r--r--r-- 1 root root 0 Jul 15 14:23 /proc/kallsyms
```

####	内核链接脚本：vmlinux.lds.S
内核链接脚本`vmlinux.lds.S`与kallsyms文件（内核符号表）之间的关系是编译阶段的协作关系，共同确保内核运行时能正确解析符号地址

1、编译流程中的依赖关系，`vmlinux.lds.S`作为链接脚本，定义了内核镜像 vmlinux的内存布局，包括代码段（`.text`）、数据段（`.data`）、初始化段（`__initcall`）等的起始/结束地址及对齐规则。即`vmlinux.lds.S`定义的布局决定了符号的物理地址，而kallsyms工具依赖这些地址生成符号表

2、内存布局中的协作，`vmlinux.lds.S`预留了符号表空间，控制内核各段的内存布局，确保代码/数据位于正确虚拟地址

3、运行时符号解析，当内核启动后，`/proc/kallsyms`通过上述数组动态解析符号地址与名称的映射关系

4、`vmlinux.lds.S`与`kallsyms`本质关系，`vmlinux.lds.S`是地基，定义内核内存布局，确定符号物理地址，而`kallsyms`是地图，用于基于地址布局生成符号名称映射，支持运行时调试

通过 vmlinux.lds.S 可以将内核的 section，大体分如下几个区间：

```TEXT
[  _text, _end )     //内核字段从 _text 入口，到 _end 结束
[  _stext, _etext )                               //文本段
[  __start_rodata, __end_rodata )                 //只读数据段
[  __init_begin, __init_end )                     //初始化段
[  __inittext_begin, __inittext_end )             //初始化文本段
[  __initdata_begin, __initdata_end )             //初始化数据段
[  _sdata, _edata )                               // 数据段，已初始化
[  __bss_start, __bss_stop )                      // bss 段，未初始化或全局变量
```

其中`.text` 段的布局如下：

```text
	.text : {			/* Real text segment		*/
		_stext = .;		/* Text and read-only data	*/
			__exception_text_start = .;
			*(.exception.text)
			__exception_text_end = .;
			IRQENTRY_TEXT
			SOFTIRQENTRY_TEXT
			ENTRY_TEXT
			TEXT_TEXT
			SCHED_TEXT
			CPUIDLE_TEXT
			LOCK_TEXT
			KPROBES_TEXT
			HYPERVISOR_TEXT
			IDMAP_TEXT
			HIBERNATE_TEXT
			TRAMP_TEXT
			*(.fixup)
			*(.gnu.warning)
		. = ALIGN(16);
		*(.got)			/* Global offset table		*/
	}
 
	. = ALIGN(SEGMENT_ALIGN);
	_etext = .;			/* End of text section */
```

####    符号表：kallsyms
每个符号条目包含以下信息：

-	地址：符号在内核地址空间中的虚拟地址
-	类型：符号的类型（如 `T` 表示函数，`D` 表示全局变量）
-	名称：符号的标识符

符号表的分类：

-	静态符号表（`System.map`）：编译时固定生成，仅包含内核核心符号。
-	动态符号表（`/proc/kallsyms`）：运行时动态生成，包含所有符号（内核核心 + 已加载模块的符号）

![]()

```bash
[root@VM-X-X-centos ~]# cat /proc/kallsyms |grep sys_call_table
ffffffff82200240 D sys_call_table
ffffffff82201260 D ia32_sys_call_table

[root@VM-X-X-centos ~]# grep "call_table" /boot/System.map-$(uname -r)
ffffffff81ba0d90 t brnf_sysctl_call_tables
ffffffff82200240 D sys_call_table
ffffffff82201260 D ia32_sys_call_table
```

对于上一小节提到的文本段`[_stext,_etext)`，看看实际数据是什么：

```BASH
[root@VM-X-X-centos edriver]# cat /proc/kallsyms |grep -A 20 _stext
ffffffff81000000 T _stext
ffffffff81000000 T _text
ffffffff81000030 T secondary_startup_64
ffffffff810000f0 T verify_cpu
ffffffff810001f0 T start_cpu0
ffffffff81000200 T __startup_64
ffffffff810005f0 T pvh_start_xen
ffffffff81000670 T __startup_secondary_64
ffffffff81000680 t trace_initcall_finish_cb
ffffffff810006c0 t perf_trace_initcall_level
ffffffff810007f0 t perf_trace_initcall_start
ffffffff810008c0 t perf_trace_initcall_finish
ffffffff810009a0 t trace_event_raw_event_initcall_level
ffffffff81000a80 t trace_raw_output_initcall_level
ffffffff81000ad0 t trace_raw_output_initcall_start
ffffffff81000b20 t trace_raw_output_initcall_finish
ffffffff81000b70 t __bpf_trace_initcall_level
......

[root@VM-X-X-centos edriver]# cat /proc/kallsyms |grep -B 10 _etext
ffffffff81e02530 T smp_error_interrupt
ffffffff81e026d0 T smp_irq_move_cleanup_interrupt
ffffffff81e02788 T __irqentry_text_end
ffffffff82000000 T __do_softirq
ffffffff82000000 T __softirqentry_text_start
ffffffff820002d7 T __softirqentry_text_end
ffffffff82000fbb t .E_read_words
ffffffff82000fbe t .E_leading_bytes
ffffffff82000fc0 t .E_trailing_bytes
ffffffff82000fc7 t .E_write_words
ffffffff82000fdc T _etext
```

再看下内核函数`do_sys_open`的地址，位于`[_stext,_etext)`之间：

```BASH
[root@VM-X-X-centos edriver]# cat /proc/kallsyms |grep -B 50000 _etext|grep do_sys_open
ffffffff81297210 T do_sys_open
ffffffff81bd61c5 t do_sys_open.cold.24
```

####    kallsyms：内核地址
在eBPF开发中，经常需要访问`/proc/kallsyms`文件来获取kprobe函数信息，`/proc/kallsyms` 中的符号地址是内核虚拟地址空间中的地址，而非进程用户空间的虚拟地址，对于`perf`、`ftrace`、`kprobe` 等工具也会依赖 `/proc/kallsyms` 解析内核地址（如 `perf report` 将地址转换为函数名）

`/proc/kallsyms`是 Linux 内核开启 `CONFIG_KALLSYMS`选项时由内核自动创建的一个虚拟文件，用于列出内核已编译进去的全部函数和变量等符号信息，简单理解`kallsyms`是一份**内核函数和变量名->虚拟内存地址的映射表**

```BASH
[root@VM-X-X-tencentos ebpf-pro]# cat /proc/kallsyms |grep do_sys_open
ffffffff813eae80 t __pfx_do_sys_openat2
ffffffff813eae90 t do_sys_openat2
ffffffff813eb8b0 T __pfx_do_sys_open
ffffffff813eb8c0 T do_sys_open
```

如上示例中，符号 `do_sys_open` 的地址 `ffffffff813eb8c0` 属于内核空间高位地址范围，由于进程用户空间是互相隔离的，每个进程拥有独立的用户空间（`32`位为 `0x00000000-0xBFFFFFFF`，`64`位为 `0x0000000000000000-0x00007FFFFFFFFFFF`），但所有进程共享同一份内核空间映射。`/proc/kallsyms` 的符号地址对所有进程是全局一致的。此外，虽然内核空间映射到每个进程的虚拟地址空间（通过页表），但进程在用户态是无权访问内核地址的，仅当进程通过系统调用陷入内核态时，才能访问内核空间

##	0x0	高端内存



##  0x0  参考
-   [4.6 深入理解 Linux 虚拟内存管理](https://www.xiaolincoding.com/os/3_memory/linux_mem.html)
-   [4.7 深入理解 Linux 物理内存管理](https://www.xiaolincoding.com/os/3_memory/linux_mem2.html#_6-1-%E5%8C%BF%E5%90%8D%E9%A1%B5%E7%9A%84%E5%8F%8D%E5%90%91%E6%98%A0%E5%B0%84)
-   [mmap 源码分析](https://leviathan.vip/2019/01/13/mmap%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/)
-   [linux源码解读（十六）：红黑树在内核的应用——虚拟内存管理](https://www.cnblogs.com/theseventhson/p/15820092.html)
-   [图解 Linux 虚拟内存空间管理](https://github.com/liexusong/linux-source-code-analyze/blob/master/process-virtual-memory-manage.md)
-   [/proc/kallsyms 全面解析和实战应用指南](https://blog.csdn.net/Interview_TC/article/details/148256969)