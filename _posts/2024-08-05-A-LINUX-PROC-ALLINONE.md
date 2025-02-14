---
layout:     post
title:      Linux proc 知识点汇总
subtitle:   
date:       2024-08-05
author:     pandaychen
header-img:
catalog: true
tags:
    - Linux
---

##  0x00    前言
`proc`文件系统（procfs）是一种基于内核的VFS，以文件系统目录和文件形式，提供一个指向内核数据结构的接口，通过它能够查看和改变各种系统属性

![]()

-   procfs 它不依赖物理存储设备，而是由内核动态生成文件和目录，用于暴露系统信息（如进程状态、硬件配置、内核参数等），它通过注册到 VFS 的机制，将自己挂载到文件系统树（`/proc`），并遵循 VFS 的接口规范（如`inode_operations`、`file_operations`）
-   内核数据结构：procfs 的内容（如 `/proc/cpuinfo`、`/proc/meminfo`等）由内核动态生成，通过回调函数响应读/写操作
-   VFS 接口：procfs 需要实现 VFS 定义的操作函数（例如 `open()`、`read()`、`readdir()`），才能被用户空间访问
-   procfs 的功能和内容完全由内核管理，通过 `mount -t proc proc /proc` 命令将 procfs 挂载到 VFS 树中，使其成为文件系统的一部分


##  0x0  参考
-   [Linux进程网络流量统计方法及实现](https://zhuanlan.zhihu.com/p/49981590)
-   [使用 golang gopacket 实现进程级流量监控](https://github.com/rfyiamcool/notes/blob/main/netflow.md)
-   [从内核代码角度详解proc目录](https://blog.spoock.com/2019/10/26/proc-from-kernel/)
-   [Linux下/proc目录简介](https://blog.spoock.com/2019/10/08/proc/)
-   [Linux Procfs (一) /proc/* 文件实例解析](https://juejin.cn/post/7055321925463048228)