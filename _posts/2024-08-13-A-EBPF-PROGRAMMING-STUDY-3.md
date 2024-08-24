---
layout:     post
title:      golang eBPF 开发入门（二）
subtitle:	kprobe/uprobe/tracepoint 开发实践
date:       2024-08-13
author:     pandaychen
catalog:    true
tags:
    - eBPF
    - Golang
---


##  0x00    前言

![programming-practice](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/ebpf/ebpf-tracing.png)

本文专注于最右侧的技术

##  0x01  kprobe/uprobe/tracepoint 技术

####    tracepoint



##  0x02  kprobe 基础实践

####  开发步骤
1、准备对应版本的 kernel 内核源码，**在对系统调用植入探针的过程中，需要了解对应函数的参数和返回值定义**

```BASH
uname -a #5.15.0-83-generic
# 确认目标设备的 linux 版本
# 下载 kernel 源码
git clone git@github.com:torvalds/linux.git   -b v5.15

# 搜索源码定义
grep -w do_unlinkat -r ./
```

2、kprobe ebpf 程序实例，这里以内核的 `do_unlinkat`（删除文件事件）进行自定义代码植入

首先，观察内核源码中 `do_unlinkat` 函数定义，如下所示。该函数接受 `2` 个参数：`dfd`（文件描述符）和 `name`（文件名结构体指针），返回值为 int，接下来编写 ebpf 程序时会用到这个定义

```C
int do_unlinkat(int dfd, struct filename *name);
```

3、编写 ebpf 探测函数 `kprobe.bpf.c` 程序（内核态代码），使用 kprobe（内核探针）在内核 `do_unlinkat` 函数的入口和退出位置添加钩子，实现对 Linux 内核中的 unlink 系统调用的入口参数和返回值的监测和捕获：

```C
#include "vmlinux.h"
#include <bpf/bpf_helpers.h>
#include <bpf/bpf_tracing.h>
#include <bpf/bpf_core_read.h>

// 定义许可证，以允许程序在内核中运行
char LICENSE[] SEC("license") = "Dual BSD/GPL";

// 定义一个名为 do_unlinkat 的 kprobe，当进入 do_unlinkat 函数时，它会被触发
SEC("kprobe/do_unlinkat")
int BPF_KPROBE(do_unlinkat, int dfd, struct filename *name) // 捕获函数的参数：dfd（文件描述符）和 name（文件名结构体指针）
{
    pid_t pid;
    const char *filename;

    // 获取当前进程的 PID（进程标识符）
    pid = bpf_get_current_pid_tgid()>> 32;
    // 读取文件名
    filename = BPF_CORE_READ(name, name);
    // 使用 bpf_printk 函数在内核日志中打印 PID 和文件名
    bpf_printk("KPROBE ENTRY pid = %d, filename = %s\n", pid, filename);
    return 0;
}

// 定义一个名为 do_unlinkat_exit 的 kretprobe，当从 do_unlinkat 函数退出时，它会被触发
SEC("kretprobe/do_unlinkat")
int BPF_KRETPROBE(do_unlinkat_exit, long ret) // 捕获函数的返回值（ret）
{
    pid_t pid;

    // 获取当前进程的 PID（进程标识符）
    pid = bpf_get_current_pid_tgid()>> 32;
    // 使用 bpf_printk 函数在内核日志中打印 PID 和返回值
    bpf_printk("KPROBE EXIT: pid = %d, ret = %ld\n", pid, ret);
    return 0;
}
```

内核态代码的要点：
-	`BPF_CORE_READ` 宏的作用
-	hook 内核函数的调用顺序

4、加载程序开发，编写一个 bpf 用户空间程序，用于加载上述 ebpf 钩子到内核空间：

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

	/* 设置 libbpf 错误和调试信息回调 */
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

	printf("Successfully started! Please run `sudo cat /sys/kernel/debug/tracing/trace_pipe`"
	       "to see output of the BPF programs.\n");

	while (!stop) {
		fprintf(stderr, ".");
		sleep(1);
	}

cleanup:
	/* 销毁挂载的 ebpf 程序 */
	kprobe_bpf__destroy(skel);
	return -err;
}
```

5、 使用 ring buffer 向用户态传递数据，上述代码中，仅仅使用 `bpf_printk` 即内核捕获到的数据打印到了内核 log（可以通过 `/sys/kernel/debug/tracing/trace_pipe` 查看）中，修改为 ringbuff 让用户态程序获取，将捕获到的数据通过 ring buffer 从内核空间传递到用户空间，当用户空间获取数据后，可以再进行后续的数据存储、处理和分析


步骤 1：定义一个 `kprobe.h` 头文件，方便内核空间和用户空间的程序使用同一个数据存储结构

```CPP
#ifndef __KPROBE_H
#define __KPROBE_H

#define MAX_FILENAME_LEN 256

struct event {
    int pid;
    char filename[MAX_FILENAME_LEN];
    bool exit_event;
    unsigned exit_code;
    unsigned long long ns;
};

#endif
```

步骤 2：修改内核态代码，定义 map，实现内核 ebpf 存储数据到 ring buffer

```CPP
#include <string.h>
#include "kprobe.h"

// 定义一个名为 rb 的 ring buffer 类型的 Map
struct {
    __uint(type, BPF_MAP_TYPE_RINGBUF);
    __uint(max_entries, 256 * 1024);    // 256 KB
} rb SEC(".maps");

//hook
SEC("kprobe/do_unlinkat")
int BPF_KPROBE(do_unlinkat, int dfd, struct filename *name)  // 该函数接受两个参数：dfd（文件描述符）和 name（文件名结构体指针）
{
    //...
    struct event *e;

    //...
    // 预订一个 ringbuf 样本空间
    e = bpf_ringbuf_reserve(&rb, sizeof(*e), 0);
    if (!e)
        return 0;
    // 设置数据
    e->pid = pid;

	// 在 ebpf 探针函数中保存数据到 ring buffer
    bpf_probe_read_str(&e->filename, sizeof(e->filename), (void *)filename);
	e->exit_event = false;
    e->ns = bpf_ktime_get_ns();
    // 提交到 ringbuf 用户空间进行后处理
    bpf_ringbuf_submit(e, 0);

    return 0;
}
```

步骤 3：用户空间读取 ring buffer，基本流程如下：

1.	定义 ring_buffer 结构体、`handle_event` 回调函数
2.	使用 `ring_buffer__new(bpf_map__fd(skel->maps.rb), handle_event, NULL, NULL)`（`skel->maps.rb` 的 `fd`）初始化用户空间缓冲区对象
3.	使用 `ring_buffer__poll` 获取内核 ebpf 程序传递的数据，收到的内核数据在 `handle_event` 回调函数中进行打印、存储、分析等后续处理

```CPP
#include <time.h>
#include "kprobe.h"

...

// ring buffer data process
static int handle_event(void *ctx, void *data, size_t data_sz)
{
	const struct event *e = data;
	struct tm *tm;
	char ts[32];
	time_t t;

	time(&t);
	tm = localtime(&t);
	strftime(ts, sizeof(ts), "%H:%M:%S", tm);

	if (e->exit_event) {
		printf("%-8s %-5s %-16s %-7d [%u]", ts, "EXIT", e->filename, e->pid, e->exit_code);
		if (e->ns)
			printf("(%llums)", e->ns / 1000000);
		printf("\n");
	} else {
		printf("%-8s %-5s %-16s %-7d %s\n", ts, "EXEC", e->filename, e->pid, e->filename);
	}

	return 0;
}

int main(int argc, char **argv)
{
	//...
	struct ring_buffer *rb = NULL;

	/* 设置环形缓冲区轮询 */
	rb = ring_buffer__new(bpf_map__fd(skel->maps.rb), handle_event, NULL, NULL);
	if (!rb) {
		err = -1;
		fprintf(stderr, "Failed to create ring buffer\n");
		goto cleanup;
	}

	/* 处理收到的内核数据 */
	printf("%-8s %-5s %-16s %-7s %s\n", "TIME", "EVENT", "FILENAME", "PID", "FILENAME/RET");
	while (!stop) {
		// 轮询内核数据
		err = ring_buffer__poll(rb, 100 /* timeout, ms */);
		if (err == -EINTR) {	/* Ctrl-C will cause -EINTR */
			err = 0;
			break;
		}
		if (err < 0) {
			printf("Error polling perf buffer: %d\n", err);
			break;
		}
	}
	// while (!stop) {
	// fprintf(stderr, ".");
	// sleep(1);
	// }
	//...
}
```

##	0x03	内核态实践（经典示例）

####    实践1：
主要介绍如何实现一个 eBPF 工具，捕获进程发送信号的系统调用集合，使用 hash map 保存状态


####	实践2：
[](https://eunomia.dev/zh/tutorials/11-bootstrap/#ebpf-bootstrapbpfc)

##	0x04	综合实践

####    捕获process启动/退出事件
本小结主要通过探测内核的`sys_enter_execve`、`sched_process_exit`事件，捕获进程的拉起和退出事件，并通过ring buffer输出到用户空间程序中，依然分为如下3个步骤：

1、步骤一，定义结构

```c
struct trace_entry {
	short unsigned int type;
	unsigned char flags;
	unsigned char preempt_count;
	int pid;
};

/* sched_process_exec tracepoint context */
struct trace_event_raw_sched_process_exec {
	struct trace_entry ent;
	unsigned int __data_loc_filename;
	int pid;
	int old_pid;
	char __data[0];
};

#define TASK_COMM_LEN 16
#define MAX_FILENAME_LEN 512

/* definition of a sample sent to user-space from BPF program */
struct event {
	int pid;
	char comm[TASK_COMM_LEN];
	char filename[MAX_FILENAME_LEN];
};
```


2、步骤二，实现内核态代码

```c

// 定义ring buffer Map
struct {
    __uint(type, BPF_MAP_TYPE_RINGBUF);
    __uint(max_entries, 256 * 1024);    // 256 KB
} rb SEC(".maps");


// 捕获进程执行事件，使用 ring buffer 向用户态打印输出
SEC("tracepoint/syscalls/sys_enter_execve")
int snoop_process_start(struct trace_event_raw_sys_enter* ctx)
{
    u64 id;
    pid_t pid;
    struct event *e;
    struct task_struct *task;

    // 获取当前进程的用户ID
    uid_t uid = (u32)bpf_get_current_uid_gid();
    // 获取当前进程ID
    id = bpf_get_current_pid_tgid();
    pid = id >> 32;
    // 获取当前进程的task_struct结构体
    task = (struct task_struct*)bpf_get_current_task();
    // 读取进程名称
    char *cmd = (char *) BPF_CORE_READ(ctx, args[0]);

    // 预订一个ringbuf样本空间
    e = bpf_ringbuf_reserve(&rb, sizeof(*e), 0);
    if (!e)
        return 0;
    // 设置数据
    e->pid = pid;
    e->uid = uid;
    e->ppid = BPF_CORE_READ(task, real_parent, pid);
    bpf_probe_read_str(&e->cmd, EXEC_CMD_LEN, cmd);
    e->ns = bpf_ktime_get_ns();
    // 提交到ringbuf用户空间进行后处理
    bpf_ringbuf_submit(e, 0);

    // 使用bpf_printk函数在内核日志中打印 PID 和文件名
    // bpf_printk("TRACEPOINT EXEC pid = %d, uid = %d, cmd = %s\n", pid, uid, e->cmd);
    return 0;
}

// 监控进程退出事件，使用 ring buffer 向用户态打印输出
SEC("tp/sched/sched_process_exit")
int snoop_process_exit(struct trace_event_raw_sched_process_template* ctx)
{
    struct task_struct *task;
    struct event *e;
    pid_t pid, tid;
    u64 id, ts, *start_ts, start_time = 0;

    // 获取当前进程的用户ID
    uid_t uid = (u32)bpf_get_current_uid_gid();
    // 获取当前进程/线程ID
    id = bpf_get_current_pid_tgid();
    pid = id >> 32;
    tid = (u32)id;
    // 获取当前进程的task_struct结构体
    task = (struct task_struct *)bpf_get_current_task();
    start_time = BPF_CORE_READ(task, start_time);

    /* ignore thread exits */
    if (pid != tid)
        return 0;

    // 预订一个ringbuf样本空间
    e = bpf_ringbuf_reserve(&rb, sizeof(*e), 0);
    if (!e)
        return 0;
    // 设置数据
    e->ns = bpf_ktime_get_ns() - start_time;
    e->pid = pid;
    e->uid = uid;
    e->ppid = BPF_CORE_READ(task, real_parent, tgid);
    e->is_exit = true;
    e->retval = (BPF_CORE_READ(task, exit_code) >> 8) & 0xff;
    bpf_get_current_comm(&e->cmd, sizeof(e->cmd));
    // 提交到ringbuf用户空间进行后处理
    bpf_ringbuf_submit(e, 0);

    // 使用bpf_printk函数在内核日志中打印 PID 和文件名
    // bpf_printk("TRACEPOINT EXIT pid = %d, uid = %d, cmd = %s\n", pid, uid, e->cmd);
    return 0;
}
```


3、步骤三，实现用户态代码

```c

int libbpf_print_fn(enum libbpf_print_level level, const char *format, va_list args)
{
	/* Ignore debug-level libbpf logs */
	if (level > LIBBPF_INFO)
		return 0;
	return vfprintf(stderr, format, args);
}

// Control-C process
static volatile bool exiting = false;
static void sig_handler(int sig)
{
	exiting = true;
}

// ring buffer data process
static int handle_event(void *ctx, void *data, size_t data_sz)
{
	const struct event *e = (struct event *)data;
    struct tm *tm;
    char ts[32];
    time_t t;

    time(&t);
    tm = localtime(&t);
    strftime(ts, sizeof(ts), "%H:%M:%S", tm);

    if (e->is_exit) {
        printf("%s %-5s %d %d %s %d %llums\n", ts, "EXIT", e->pid, e->uid, e->cmd, e->retval, e->ns / 1000000);
    } else {
        printf("%s %-5s %d %d %d  %s\n", ts, "EXEC", e->pid, e->ppid, e->uid, e->cmd);
    }

    return 0;
}

int main(int argc, char **argv)
{
    struct exec_bpf *skel;
    int err;
    struct ring_buffer *rb = NULL;

    /* 设置libbpf错误和调试信息回调 */
    libbpf_set_print(libbpf_print_fn);

    /* Control-C 停止信号 */
	signal(SIGINT, sig_handler);
	signal(SIGTERM, sig_handler);

    /* 加载并验证 exec.bpf.c 应用程序 */
    skel = exec_bpf__open_and_load();
    if (!skel) {
        fprintf(stderr, "Failed to open BPF skeleton\n");
        return 1;
    }

    /* 附加 exec.bpf.c 程序到跟踪点 */
    err = exec_bpf__attach(skel);
    if (err) {
        fprintf(stderr, "Failed to attach BPF skeleton\n");
        exec_bpf__destroy(skel);
        return -err;
    }
    // printf("Successfully started! Please run `sudo cat /sys/kernel/debug/tracing/trace_pipe` to see output of the BPF programs.\n");

    /* 设置环形缓冲区轮询 */
    rb = ring_buffer__new(bpf_map__fd(skel->maps.rb), handle_event, NULL, NULL);
    if (!rb) {
        err = -1;
        fprintf(stderr, "Failed to create ring buffer\n");
        exec_bpf__destroy(skel);
        return -err;
    }

    /* 处理收到的内核数据 */
    printf("%-8s %-8s %-7s %-7s %-16s %-8s %-8s\n", "TIME", "TYPE", "PID", "UID", "CMD", "RET", "DURATION");
    while (!exiting) {
        // 轮询内核数据
        err = ring_buffer__poll(rb, 100 /* timeout, ms */);
        if (err == -EINTR) {    /* Ctrl-C will cause -EINTR */
            err = 0;
            break;
        }
        if (err < 0) {
            printf("Error polling perf buffer: %d\n", err);
            break;
        }
    }
}
```

##	0x0	How to Debug


##	0x0	teleport 的 ebpf 应用
teleport 基于 [ebpf](https://github.com/gravitational/teleport/tree/master/bpf/enhancedrecording) 也实现对 ssh 命令审计的增强功能，预期可以实现下面几种特殊场景的捕获（原始基于键盘输入及屏显输出，有局限性）：

-	混淆命令（Obfuscation）：如经典用例 `echo Y3VybCBodHRwOi8vd3d3LmV4YW1wbGUuY29tCg== | base64 -d | sh`，最终运行的结果是 `curl example.com`，但是当下方法只能记录到 `echo xxx` 的部分
-	Shell 脚本审计（脚本内的命令）：如果用户通过上传并执行脚本的方式，也不能捕获脚本中运行的命令，只能捕获脚本的输出
-	终端控制（Terminal controls）：比如执行了 `stty -echo` 之后，从命令行输入的命令就不再回显了，如果运行了补全命令 `ifc<TAB>`，那么执行如下图

![stty](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/teleport/ebpf/stty-command.png)

####	teleport 的方案

![teleport](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/teleport/ebpf/ssh-audit-1.jpg)

Teleport 实现了如下 `3` 个 BPF hooks:
-	`execsnoop` 捕获程序执行
-	`opensnoop` 捕获程序打开的文件
-	`tcpconnect` 捕获程序建立的 TCP 连接

此外，teleport 还实现了会话与 ebf hooks 的捕获结果做了关联，其原理是使用了 `cgroups`（`cgroupv2`）。Teleport 启动 SSH 会话时，它将首先重新启动自身并将其置于 `cgroup` 中，这不仅允许该进程，而且可以使用唯一 ID 跟踪 Teleport 启动的所有将来的进程。Teleport 运行的 BPF 程序已更新，还可以发出执行它们的程序的 cgroup ID，这样就可以将事件与特定的 SSH 会话和身份相关联，参考 [原文](https://goteleport.com/blog/enhanced-session-recording/)

##	0x0	epbf 开发要点（汇总）

####	再看CO-RE
下面三个函数的用途和区别是什么？

-	`BPF_CORE_READ`
-	`bpf_probe_read`
-	`bpf_probe_read_str`


####	how to find hooks

##  0x0    参考
- [eBPF—使用 kprobe 探测内核系统调用](https://blog.yanjingang.com/?p=8062)
- [About Scaffolding for BPF application development with libbpf and BPF CO-RE](https://github.com/libbpf/libbpf-bootstrap)
- [调试你的 BPF 程序](https://davidlovezoe.club/wordpress/archives/963)
- [实战 eBPF kprobe 函数插桩](https://www.cnxct.com/using-ebpf-kprobe-to-file-notify/)
- [eBPF apps with libbpf](https://github.com/feiskyer/ebpf-apps/tree/main/bpf-apps)
- [[译] BPF CO-RE 参考指南 (2021)](https://mozillazg.com/2024/07/bpf-core-reference-guide-zh.html)
- [ebpf user-space probes 原理探究](https://www.edony.ink/deep-in-ebpf-uprobe/)
- [如何使用 eBPF 进行追踪](https://kiosk007.top/post/%E5%A6%82%E4%BD%95%E4%BD%BF%E7%94%A8ebpf%E8%BF%9B%E8%A1%8C%E8%BF%BD%E8%B8%AA/)
- [gobpf 使用示例：如何找到一个系统调用对应的可用于 kprobe SEC 的内核函数](https://mozillazg.com/2021/05/ebpf-gobpf-how-to-find-system-call-kernel-function-for-kprobe-elf-section.html)
- [eBPF-tracing 使用 eBPF 编写监控类程序](https://p1nant0m.com/2022-07-25-ebpf-tracing/)
- [eBPF 入门开发实践教程六：捕获进程发送信号的系统调用集合，使用 hash map 保存状态](https://eunomia.dev/zh/tutorials/6-sigsnoop/#sigsnoop)