---
layout:     post
title:  Linux 内核之旅（七）：虚拟内存管理（下）TODO
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

####    kallsyms

```BASH
[root@VM-X-X-tencentos ebpf-pro]# cat /proc/kallsyms |grep do_sys_open
ffffffff813eae80 t __pfx_do_sys_openat2
ffffffff813eae90 t do_sys_openat2
ffffffff813eb8b0 T __pfx_do_sys_open
ffffffff813eb8c0 T do_sys_open
```

##  0x0  参考
-   [4.6 深入理解 Linux 虚拟内存管理](https://www.xiaolincoding.com/os/3_memory/linux_mem.html)
-   [4.7 深入理解 Linux 物理内存管理](https://www.xiaolincoding.com/os/3_memory/linux_mem2.html#_6-1-%E5%8C%BF%E5%90%8D%E9%A1%B5%E7%9A%84%E5%8F%8D%E5%90%91%E6%98%A0%E5%B0%84)
-   [mmap 源码分析](https://leviathan.vip/2019/01/13/mmap%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/)
-   [linux源码解读（十六）：红黑树在内核的应用——虚拟内存管理](https://www.cnblogs.com/theseventhson/p/15820092.html)
-   [图解 Linux 虚拟内存空间管理](https://github.com/liexusong/linux-source-code-analyze/blob/master/process-virtual-memory-manage.md)
-   [/proc/kallsyms 全面解析和实战应用指南](https://blog.csdn.net/Interview_TC/article/details/148256969)