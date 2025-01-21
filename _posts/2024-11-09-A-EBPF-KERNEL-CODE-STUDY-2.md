---
layout:     post
title:      EBPF 内核态代码学习（二）：使用 eBPF 隐藏进程 / 文件信息
subtitle:   理解 ps/ls 等运行原理
date:       2024-11-09
author:     pandaychen
header-img:
catalog: true
tags:
    - eBPF
---

##  0x00    前言
在主机安全对抗中，有一项技术叫进程隐藏，即能让特定的进程对 os 的常规检测机制变得不可见，其基本原理是 Linux 系统的 VFS，每个进程都在 `/proc/` 目录下有一个以其进程 ID 命名的子目录，其中包含了该进程的各种信息（`ps` 命令就是通过查找这些文件夹来显示进程信息的，`ls` 命令也是同样原理）

-   进程隐藏：如果能隐藏某个进程的 `/proc/${id}` 目录，就能让这个进程对 `ps` 的搜索失效
-   文件隐藏

实现上述两种场景，核心就是两点：
-   hook 系统调用： `getdents64` 系统调用可以读取目录下的文件信息，可以通过挂接这个系统调用，修改它返回的结果，从而达到隐藏文件的目的
-   借助于 eBPF `bpf_probe_write_user` 功能，该函数可以 ** 修改用户空间的内存 **，因此能用来修改 `getdents64` 系统调用返回的结果（注意：内核空间是无法修改的）

####    bpf_probe_write_user 方法
[`bpf_probe_write_user`](https://docs.ebpf.io/linux/helper-function/bpf_probe_write_user/)，该函数允许 eBPF 程序写入当前正在运行的进程的用户空间内存，因此基于 ebpf 实现的恶意软件可以使用此功能在 hook 系统调用运行期间修改进程的内存，例如一个典型的恶意场景是修改 `sudoer` 文件，参考 [使用 eBPF 添加 sudo 用户](https://eunomia.dev/zh/tutorials/26-sudo/)

该恶意命令的工作原理是：通过拦截 `sudo` 读取 `/etc/sudoers` 文件这个过程，并将第一行覆盖为 `<username> ALL=(ALL:ALL) NOPASSWD:ALL #` 的方式工作，这一过程有点像欺骗了 `sudo`，使其认为这个低权限用户被允许成为 `root`。其他程序如 `cat` 或 `sudoedit` 不受到影响，因为对于这些程序来说，实际 `/etc/sudoers` 文件内容并未改变，低权限用户并没有这些权限，只是恶意程序修改了返回的用户态内存（行尾的 `#` 确保行的其余部分被当作注释处理，因此并不会破坏文件的逻辑）

##  0x01   getdents64 的实现
`getdents64`​系统调用，用于扫描并读取指定目录的目录项信息。它的功能是从指定的目录中读取目录项的详细信息，包括文件名、文件类型、文件的 `inode` 号等，`64` 表示支持大文件（`64` 位文件偏移），每次调用该函数时，它会读取尽可能多的目录项，直到 `dirent​` 缓冲区填满或者目录已经读取完毕

```CPP
//fd​：文件描述符，通常是一个已经打开的目录文件的文件描述符
​//dirent​：一个用于存储目录项信息的缓冲区，格式化后的，也可以理解为一个 long unsigned int*
​//count​：dirent​ 缓冲区的大小，以字节为单位

// 返回值：
// 成功时，返回读取的字节数
// 如果已到达目录的末尾（没有更多的目录项可读），返回 0
// 出错时，返回 -1，并设置 errno​ 变量来指示错误的类型
int getdents64(unsigned int fd, struct linux_dirent64 *dirp, unsigned int count);
```

这里要了解，参数 `dirp` 并不是仅仅指向一个 `linux_dirent64` 结构，而是一个 index 数组链表，一个用于存储目录项 dentry 信息的缓冲区

其中，`struct linux_dirent64`（目录项）的定义如下，结构体包含了目录项的各种属性，如文件名、文件类型、inode 号等

```CPP
struct linux_dirent64 {
     u64        d_ino;    /* 64-bit inode number */
     u64        d_off;    /* 64-bit offset to next structure */
     unsigned short d_reclen; /* Size of this dirent */
     unsigned char  d_type;   /* File type */
     char           d_name[]; /* Filename (null-terminated) */
};
```

##  0x02    ebpf 的 hook 实现
要实现的目的如下，假设黄色部分是需要隐藏的，那么 ebpf hook 要做的事情是在调用 `getdents64` 时，找到这个部分并且跳过它

![dirent](https://github.com/pandaychen/pandaychen.github.io/blob/master/blog_img/ebpf/dev/hidepid/ebpf-hide-pid-1.png?raw=true)

####    MAPS 定义

```CPP
// SPDX-License-Identifier: BSD-3-Clause
#include "vmlinux.h"
#include <bpf/bpf_helpers.h>
#include <bpf/bpf_tracing.h>
#include <bpf/bpf_core_read.h>
#include "common.h"

char LICENSE[] SEC("license") = "Dual BSD/GPL";

// Ringbuffer Map to pass messages from kernel to user
struct {
    __uint(type, BPF_MAP_TYPE_RINGBUF);
    __uint(max_entries, 256 * 1024);
} rb SEC(".maps");

// Map to fold the dents buffer addresses
struct {
    __uint(type, BPF_MAP_TYPE_HASH);
    __uint(max_entries, 8192);
    __type(key, size_t);
    __type(value, long unsigned int);
} map_buffs SEC(".maps");

// Map used to enable searching through the
// data in a loop
struct {
    __uint(type, BPF_MAP_TYPE_HASH);
    __uint(max_entries, 8192);
    __type(key, size_t);
    __type(value, int);
} map_bytes_read SEC(".maps");

// Map with address of actual
struct {
    __uint(type, BPF_MAP_TYPE_HASH);
    __uint(max_entries, 8192);
    __type(key, size_t);
    __type(value, long unsigned int);
} map_to_patch SEC(".maps");

// Map to hold program tail calls
struct {
    __uint(type, BPF_MAP_TYPE_PROG_ARRAY);
    __uint(max_entries, 5);
    __type(key, __u32);
    __type(value, __u32);
} map_prog_array SEC(".maps");
```

####    内核态实现


####    用户态实现

##  0x03  参考
-   [eBPF 开发实践：使用 eBPF 隐藏进程或文件信息](https://eunomia.dev/zh/tutorials/24-hide/)
-   [sudoadd](https://github.com/eunomia-bpf/bpf-developer-tutorial/blob/main/src/26-sudo/sudoadd.bpf.c)