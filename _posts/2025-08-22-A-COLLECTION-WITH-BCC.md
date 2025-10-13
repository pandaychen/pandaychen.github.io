---
layout:     post
title:      BCC
subtitle:   BCC all in one
date:       2025-06-22
author:     pandaychen
header-img:
catalog: true
tags:
    - eBPF
    - BCC
---

##  0x00    前言

参考：
-   [BCC工具示例1](https://github.com/iovisor/bcc/tree/master/tools/old)
-   [BCC工具示例2](https://github.com/iovisor/bcc/tree/master/tools)

##  0x01    基础

##  0x02    细节

####    对多内核版本的兼容
以[`filegone`](https://github.com/iovisor/bcc/blob/master/tools/filegone.py)工具为例，因为内核5.x之上`vfs_rename`函数的入参发生了变化，所以代码中需要调用`kernel_struct_has_field`进行检测

```C
//版本https://elixir.bootlin.com/linux/v6.17.1/source/fs/namei.c#L5007
int vfs_rename(struct renamedata *rd)

//版本https://elixir.bootlin.com/linux/v4.11.6/source/fs/namei.c#L4327
int vfs_rename(struct inode *old_dir, struct dentry *old_dentry,
	       struct inode *new_dir, struct dentry *new_dentry,
	       struct inode **delegated_inode, unsigned int flags)
```

```PYTHON
# check 'struct renamedata' exist or not
if BPF.kernel_struct_has_field(b'renamedata', b'new_mnt_idmap') == 1:
    bpf_text = bpf_text.replace('TRACE_VFS_RENAME_FUNC', bpf_vfs_rename_text_new)
    bpf_text = bpf_text.replace('TRACE_VFS_UNLINK_FUNC', bpf_vfs_unlink_text_3)
elif BPF.kernel_struct_has_field("renamedata", "old_mnt_userns") == 1:
    bpf_text = bpf_text.replace('TRACE_VFS_RENAME_FUNC', bpf_vfs_rename_text_new)
    bpf_text = bpf_text.replace('TRACE_VFS_UNLINK_FUNC', bpf_vfs_unlink_text_2)
else:
    bpf_text = bpf_text.replace('TRACE_VFS_RENAME_FUNC', bpf_vfs_rename_text_old)
    bpf_text = bpf_text.replace('TRACE_VFS_UNLINK_FUNC', bpf_vfs_unlink_text_1)
```

####    kernel_struct_has_field的实现

##  0x03    代码汇总：进程（调度）类

1、[runqslower](https://github.com/iovisor/bcc/blob/master/tools/runqslower.py)

内核态代码：

```PYTHON
# define BPF program
bpf_text = """
#include <uapi/linux/ptrace.h>
#include <linux/sched.h>
#include <linux/nsproxy.h>
#include <linux/pid_namespace.h>

BPF_ARRAY(start, u64, MAX_PID);

struct data_t {
    u32 pid;
    u32 prev_pid;
    char task[TASK_COMM_LEN];
    char prev_task[TASK_COMM_LEN];
    u64 delta_us;
};

BPF_PERF_OUTPUT(events);

// record enqueue timestamp
static int trace_enqueue(u32 tgid, u32 pid)
{
    if (FILTER_PID || FILTER_TGID || pid == 0)
        return 0;
    u64 ts = bpf_ktime_get_ns();
    start.update(&pid, &ts);
    return 0;
}
"""

bpf_text_kprobe = """
int trace_wake_up_new_task(struct pt_regs *ctx, struct task_struct *p)
{
    return trace_enqueue(p->tgid, p->pid);
}

int trace_ttwu_do_wakeup(struct pt_regs *ctx, struct rq *rq, struct task_struct *p,
    int wake_flags)
{
    return trace_enqueue(p->tgid, p->pid);
}

// calculate latency
int trace_run(struct pt_regs *ctx, struct task_struct *prev)
{
    u32 pid, tgid;

    // ivcsw: treat like an enqueue event and store timestamp
    if (prev->STATE_FIELD == TASK_RUNNING) {
        tgid = prev->tgid;
        pid = prev->pid;
        u64 ts = bpf_ktime_get_ns();
        if (pid != 0) {
            if (!(FILTER_PID) && !(FILTER_TGID)) {
                start.update(&pid, &ts);
            }
        }
    }

    pid = bpf_get_current_pid_tgid();

    u64 *tsp, delta_us;

    // fetch timestamp and calculate delta
    tsp = start.lookup(&pid);
    if ((tsp == 0) || (*tsp == 0)) {
        return 0;   // missed enqueue
    }
    delta_us = (bpf_ktime_get_ns() - *tsp) / 1000;

    if (FILTER_US)
        return 0;

    struct data_t data = {};
    data.pid = pid;
    data.prev_pid = prev->pid;
    data.delta_us = delta_us;
    bpf_get_current_comm(&data.task, sizeof(data.task));
    bpf_probe_read_kernel_str(&data.prev_task, sizeof(data.prev_task), prev->comm);

    // output
    events.perf_submit(ctx, &data, sizeof(data));

    //array map has no delete method, set ts to 0 instead
    *tsp = 0;
    return 0;
}
"""

bpf_text_raw_tp = """
RAW_TRACEPOINT_PROBE(sched_wakeup)
{
    // TP_PROTO(struct task_struct *p)
    struct task_struct *p = (struct task_struct *)ctx->args[0];
    return trace_enqueue(p->tgid, p->pid);
}

RAW_TRACEPOINT_PROBE(sched_wakeup_new)
{
    // TP_PROTO(struct task_struct *p)
    struct task_struct *p = (struct task_struct *)ctx->args[0];
    u32 tgid, pid;

    bpf_probe_read_kernel(&tgid, sizeof(tgid), &p->tgid);
    bpf_probe_read_kernel(&pid, sizeof(pid), &p->pid);
    return trace_enqueue(tgid, pid);
}

RAW_TRACEPOINT_PROBE(sched_switch)
{
    // TP_PROTO(bool preempt, struct task_struct *prev, struct task_struct *next)
    struct task_struct *prev = (struct task_struct *)ctx->args[1];
    struct task_struct *next= (struct task_struct *)ctx->args[2];
    u32 tgid, pid;
    long state;

    // ivcsw: treat like an enqueue event and store timestamp
    bpf_probe_read_kernel(&state, sizeof(long), (const void *)&prev->STATE_FIELD);
    bpf_probe_read_kernel(&pid, sizeof(prev->pid), &prev->pid);
    if (state == TASK_RUNNING) {
        bpf_probe_read_kernel(&tgid, sizeof(prev->tgid), &prev->tgid);
        u64 ts = bpf_ktime_get_ns();
        if (pid != 0) {
            if (!(FILTER_PID) && !(FILTER_TGID)) {
                start.update(&pid, &ts);
            }
        }

    }

    u32 prev_pid;
    u64 *tsp, delta_us;

    prev_pid = pid;
    bpf_probe_read_kernel(&pid, sizeof(next->pid), &next->pid);

    // fetch timestamp and calculate delta
    tsp = start.lookup(&pid);
    if ((tsp == 0) || (*tsp == 0)) {
        return 0;   // missed enqueue
    }
    delta_us = (bpf_ktime_get_ns() - *tsp) / 1000;

    if (FILTER_US)
        return 0;

    struct data_t data = {};
    data.pid = pid;
    data.prev_pid = prev_pid;
    data.delta_us = delta_us;
    bpf_probe_read_kernel_str(&data.task, sizeof(data.task), next->comm);
    bpf_probe_read_kernel_str(&data.prev_task, sizeof(data.prev_task), prev->comm);

    // output
    events.perf_submit(ctx, &data, sizeof(data));

    //array map has no delete method, set ts to 0 instead
    *tsp = 0;
    return 0;
}
"""
```

##  0x04   代码：网络类


##  0x05   代码：VFS类


##  0x06   代码：内存类


##  0x07 参考
-   [bpftrace一行教程](https://eunomia.dev/zh/tutorials/bpftrace-tutorial/)
-   [Linux探测工具BCC(可观测性)](https://www.cnblogs.com/charlieroro/p/13265252.html)