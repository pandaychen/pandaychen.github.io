---
layout:     post
title:      golang eBPF 开发入门（三）
subtitle:
date:       2024-08-13
author:     pandaychen
catalog:    true
tags:
    - eBPF
    - Golang
---


##  0x00    前言


##  0x01  kprobe技术


##  0x02  kprobe基础实践

####  开发步骤
1、准备对应版本的kernel内核源码，**在对系统调用植入探针的过程中，需要了解对应函数的参数和返回值定义**

```BASH
uname -a #5.15.0-83-generic
# 确认目标设备的linux版本
# 下载kernel源码
git clone git@github.com:torvalds/linux.git   -b v5.15

# 搜索源码定义
grep -w do_unlinkat -r ./
```

2、kprobe ebpf程序实例，这里以内核的`do_unlinkat`（删除文件事件）进行自定义代码植入

首先，观察内核源码中`do_unlinkat`函数定义，如下所示。该函数接受`2`个参数：`dfd`（文件描述符）和`name`（文件名结构体指针），返回值为int，接下来编写ebpf程序时会用到这个定义

```C
int do_unlinkat(int dfd, struct filename *name);
```

3、编写ebpf探测函数`kprobe.bpf.c`程序（内核态代码），使用kprobe（内核探针）在内核`do_unlinkat`函数的入口和退出位置添加钩子，实现对Linux 内核中的unlink 系统调用的入口参数和返回值的监测和捕获：

```C
#include "vmlinux.h"
#include <bpf/bpf_helpers.h>
#include <bpf/bpf_tracing.h>
#include <bpf/bpf_core_read.h>

// 定义许可证，以允许程序在内核中运行
char LICENSE[] SEC("license") = "Dual BSD/GPL";

// 定义一个名为do_unlinkat的 kprobe，当进入do_unlinkat函数时，它会被触发
SEC("kprobe/do_unlinkat")
int BPF_KPROBE(do_unlinkat, int dfd, struct filename *name) // 捕获函数的参数：dfd（文件描述符）和name（文件名结构体指针）
{
    pid_t pid;
    const char *filename;

    // 获取当前进程的 PID（进程标识符）
    pid = bpf_get_current_pid_tgid() >> 32;
    // 读取文件名
    filename = BPF_CORE_READ(name, name);
    // 使用bpf_printk函数在内核日志中打印 PID 和文件名
    bpf_printk("KPROBE ENTRY pid = %d, filename = %s\n", pid, filename);
    return 0;
}

// 定义一个名为do_unlinkat_exit的 kretprobe，当从do_unlinkat函数退出时，它会被触发
SEC("kretprobe/do_unlinkat")
int BPF_KRETPROBE(do_unlinkat_exit, long ret) // 捕获函数的返回值（ret）
{
    pid_t pid;

    // 获取当前进程的 PID（进程标识符）
    pid = bpf_get_current_pid_tgid() >> 32;
    // 使用bpf_printk函数在内核日志中打印 PID 和返回值
    bpf_printk("KPROBE EXIT: pid = %d, ret = %ld\n", pid, ret);
    return 0;
}
```

4、加载程序开发，编写一个bpf用户空间程序，用于加载上述ebpf钩子到内核空间：

```C
#include <stdio.h>
#include <unistd.h>
#include <signal.h>
#include <string.h>
#include <errno.h>
#include <sys/resource.h>
#include <bpf/libbpf.h>
#include "kprobe.skel.h"

static int libbpf_print_fn(enum libbpf_print_level level, const char *format, va_list args)
{
	return vfprintf(stderr, format, args);
}

static volatile sig_atomic_t stop;
static void sig_int(int signo)
{
	stop = 1;
}

int main(int argc, char **argv)
{
	struct kprobe_bpf *skel;
	int err;

	/* 设置libbpf错误和调试信息回调 */
	libbpf_set_print(libbpf_print_fn);

	/* 加载并验证 kprobe.bpf.c 应用程序 */
	skel = kprobe_bpf__open_and_load();
	if (!skel) {
		fprintf(stderr, "Failed to open BPF skeleton\n");
		return 1;
	}

	/* 附加 kprobe.bpf.c 程序到跟踪点 */
	err = kprobe_bpf__attach(skel);
	if (err) {
		fprintf(stderr, "Failed to attach BPF skeleton\n");
		goto cleanup;
	}

	/* Control-C 停止信号 */
	if (signal(SIGINT, sig_int) == SIG_ERR) {
		fprintf(stderr, "can't set signal handler: %s\n", strerror(errno));
		goto cleanup;
	}

	printf("Successfully started! Please run `sudo cat /sys/kernel/debug/tracing/trace_pipe` "
	       "to see output of the BPF programs.\n");

	while (!stop) {
		fprintf(stderr, ".");
		sleep(1);
	}

cleanup:
	/* 销毁挂载的ebpf程序 */
	kprobe_bpf__destroy(skel);
	return -err;
}
```

5、 使用 ring buffer 向用户态传递数据，上述代码中，仅仅使用`bpf_printk`即内核捕获到的数据打印到了内核log中，修改为ringbuff让用户态程序获取


##  0x03    参考
- [eBPF—使用kprobe探测内核系统调用](https://blog.yanjingang.com/?p=8062)