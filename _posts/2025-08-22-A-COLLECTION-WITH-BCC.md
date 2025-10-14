---
layout:     post
title:      BCC with python3
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
本小节汇总下基于python开发BCC程序的相关知识点，基于[tag-v0.35.0](https://github.com/iovisor/bcc/tree/v0.35.0)

参考：
-   [BCC工具示例1](https://github.com/iovisor/bcc/tree/master/tools/old)
-   [BCC工具示例2](https://github.com/iovisor/bcc/tree/master/tools)
-   [bcc Python Developer Tutorial](https://github.com/iovisor/bcc/blob/master/docs/tutorial_bcc_python_developer.md)

##  0x01    基础

1.  C开发BPF内核态代码
2.  在编译之前，对BPF程序进行rewrite
3.  把BPF程序编译成BPF字节码
4.  把编译出的（以及rewritten）BPF字节码加载到内核
5.  把BPF program（就是BPF程序中的函数）attach到events（例如kprobe）
6.  从BPF map storage读取BPF的输出

![BPF-Compiler-Collection]()

以`biolatency.py`的实现为例：

![BCC-implement-details.png]()

##  0x02    细节
BCC 如何解决内核兼容性问题？这里列举几个要点

1、在运行 eBPF 程序的时候，使用当前kernel安装的内核头文件进行就地编译，这样就可以确保 eBPF 程序中所引用的内核数据结构和函数签名等，跟运行中的内核是完全匹配的

2、在 eBPF 程序编译前事先探测内核支持的函数签名和数据结构，进而为 eBPF 程序生成适配当前内核的版本。比如，在块设备 I/O 延迟跟踪程序 `biolantecy` 的实现中，BCC 借助库函数 `BPF.get_kprobe_functions()` 来判断内核是不是支持特定的探针函数，进而再根据结果去选择挂载点：

```PYTHON
if BPF.get_kprobe_functions(b'__blk_account_io_start'):
    b.attach_kprobe(event="__blk_account_io_start", fn_name="trace_req_start")
  else:
    b.attach_kprobe(event="blk_account_io_start", fn_name="trace_req_start")
  
  if BPF.get_kprobe_functions(b'__blk_account_io_done'):
      b.attach_kprobe(event="__blk_account_io_done", fn_name="trace_req_done")
  else:
      b.attach_kprobe(event="blk_account_io_done", fn_name="trace_req_done")
```

`kernel_struct_has_field`的本质是读取`/proc/kallsyms`文件来判断对应的内核符号是否存在，参考实现[`get_kprobe_functions`](https://github.com/iovisor/bcc/blob/master/src/python/bcc/__init__.py#L719C9-L719C29)，注意到其入参是支持正则匹配的

3、借助`kernel_struct_has_field`判断某个内核结构体是否具有指定的字段（下文）

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

工具中对参数兼容性的处理：

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
BCC中的[`kernel_struct_has_field`](https://github.com/iovisor/bcc/blob/master/src/python/bcc/__init__.py#L1283)方法，最终会调用到libbpf的`kernel_struct_has_field`函数：

```CPP
int kernel_struct_has_field(const char *struct_name, const char *field_name)
{
  const struct btf_type *btf_type;
  struct btf *btf;
  int ret, btf_id;

  //从当前运行的内核中加载 vmlinux 的 BTF 信息
  btf = btf__load_vmlinux_btf();
  ret = libbpf_get_error(btf);
  if (ret)
    return -1;
 
  //在加载的BTF数据中，根据名称（如 "struct task_struct"）和种类（BTF_KIND_STRUCT）查找特定的结构体
  btf_id = btf__find_by_name_kind(btf, struct_name, BTF_KIND_STRUCT);
  if (btf_id < 0) {
    ret = -1;
    goto cleanup;
  }

  btf_type = btf__type_by_id(btf, btf_id);
  // 在找到的结构体类型（btf_type）中，继续查找是否包含指定名称（如 "wake_entry"）的成员（字段）
  // 这个函数（或其等效逻辑）会遍历结构体的所有成员，进行名称匹配
  ret = find_member_by_name(btf, btf_type, field_name);

cleanup:
  btf__free(btf);
  return ret;
}
```

####    BCC的rewrite
1、case

```C
pid_t pid = task->pid
```
重写成

```C
pid_t pid;
bpf_probe_read(&pid, sizeof(pid), &task->pid);
```

##  0x03    基础示例

##  0x04    代码汇总：进程（调度）类

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

##  0x05   代码：网络类


##  0x06   代码：文件类

1、ext4slower：[`ext4slower`](https://github.com/iovisor/bcc/blob/v0.35.0/tools/ext4slower.py)会跟踪ext4文件系统的读/写/打开和`fsyncs`等操作，并在超过阈值后打印信息，借此排查慢速IO问题，该工具可以在 VFS->文件系统接口层面进行跟踪，内核态代码如下：

针对四种类型`R/w/O/S`的监控路径，比较单一，仅仅是基于kprobe/kretprobe的耗时，四种类型之间没有计算关联

-   `kprobe(xxx)`：在 xxx开始时记录时间戳
-   `kretprobe(xxx)`：在 xxx结束时再次获取时间，计算出耗时，并且能够访问到xxx内核函数的返回值

```PYTHON
# define BPF program
bpf_text = """
......

// XXX: switch these to char's when supported
#define TRACE_READ      0
#define TRACE_WRITE     1
#define TRACE_OPEN      2
#define TRACE_FSYNC     3

struct val_t {
    u64 ts;
    u64 offset;
    struct file *fp;
};

struct data_t {
    // XXX: switch some to u32's when supported
    u64 ts_us;
    u64 type;
    u32 size;
    u64 offset;
    u64 delta_us;
    u32 pid;
    char task[TASK_COMM_LEN];
    char file[DNAME_INLINE_LEN];
};

BPF_HASH(entryinfo, u64, struct val_t);
BPF_PERF_OUTPUT(events);

//
// Store timestamp and size on entry
//

// The current ext4 (Linux 4.5) uses generic_file_read_iter(), instead of it's
// own function, for reads. So we need to trace that and then filter on ext4,
// which I do by checking file->f_op.
// The new Linux version (since form 4.10) uses ext4_file_read_iter(), And if the 'CONFIG_FS_DAX' 
// is not set, then ext4_file_read_iter() will call generic_file_read_iter(), else it will call
// ext4_dax_read_iter(), and trace generic_file_read_iter() will fail.
int trace_read_entry(struct pt_regs *ctx, struct kiocb *iocb)
{
    u64 id =  bpf_get_current_pid_tgid();
    u32 pid = id >> 32; // PID is higher part

    if (FILTER_PID)
        return 0;

    // ext4 filter on file->f_op == ext4_file_operations
    struct file *fp = iocb->ki_filp;
    if ((u64)fp->f_op != EXT4_FILE_OPERATIONS)
        return 0;

    // store filep and timestamp by id
    struct val_t val = {};
    val.ts = bpf_ktime_get_ns();
    val.fp = fp;
    val.offset = iocb->ki_pos;
    if (val.fp)
        entryinfo.update(&id, &val);

    return 0;
}

// ext4_file_write_iter():
int trace_write_entry(struct pt_regs *ctx, struct kiocb *iocb)
{
    u64 id = bpf_get_current_pid_tgid();
    u32 pid = id >> 32; // PID is higher part

    if (FILTER_PID)
        return 0;

    // store filep and timestamp by id
    struct val_t val = {};
    val.ts = bpf_ktime_get_ns();
    val.fp = iocb->ki_filp;
    val.offset = iocb->ki_pos;
    if (val.fp)
        entryinfo.update(&id, &val);

    return 0;
}

// ext4_file_open():
int trace_open_entry(struct pt_regs *ctx, struct inode *inode,
    struct file *file)
{
    u64 id = bpf_get_current_pid_tgid();
    u32 pid = id >> 32; // PID is higher part

    if (FILTER_PID)
        return 0;

    // store filep and timestamp by id
    struct val_t val = {};
    val.ts = bpf_ktime_get_ns();
    val.fp = file;
    val.offset = 0;
    if (val.fp)
        entryinfo.update(&id, &val);

    return 0;
}

// ext4_sync_file():
int trace_fsync_entry(struct pt_regs *ctx, struct file *file)
{
    u64 id = bpf_get_current_pid_tgid();
    u32 pid = id >> 32; // PID is higher part

    if (FILTER_PID)
        return 0;

    // store filep and timestamp by id
    struct val_t val = {};
    val.ts = bpf_ktime_get_ns();
    val.fp = file;
    val.offset = 0;
    if (val.fp)
        entryinfo.update(&id, &val);

    return 0;
}

static int trace_return(struct pt_regs *ctx, int type)
{
    struct val_t *valp;
    u64 id = bpf_get_current_pid_tgid();
    u32 pid = id >> 32; // PID is higher part

    valp = entryinfo.lookup(&id);
    if (valp == 0) {
        // missed tracing issue or filtered
        return 0;
    }

    // calculate delta
    u64 ts = bpf_ktime_get_ns();
    u64 delta_us = (ts - valp->ts) / 1000;
    // 每统计一次事件就删除
    entryinfo.delete(&id);
    if (FILTER_US)
        return 0;

    // populate output struct
    struct data_t data = {};
    data.type = type;
    data.size = PT_REGS_RC(ctx);
    data.delta_us = delta_us;
    data.pid = pid;
    data.ts_us = ts / 1000;
    data.offset = valp->offset;
    bpf_get_current_comm(&data.task, sizeof(data.task));

    // workaround (rewriter should handle file to d_name in one step):
    struct dentry *de = NULL;
    struct qstr qs = {};
    de = valp->fp->f_path.dentry;
    qs = de->d_name;
    if (qs.len == 0)
        return 0;
    bpf_probe_read_kernel(&data.file, sizeof(data.file), (void *)qs.name);

    // output
    events.perf_submit(ctx, &data, sizeof(data));

    return 0;
}

int trace_read_return(struct pt_regs *ctx)
{
    return trace_return(ctx, TRACE_READ);
}

int trace_write_return(struct pt_regs *ctx)
{
    return trace_return(ctx, TRACE_WRITE);
}

int trace_open_return(struct pt_regs *ctx)
{
    return trace_return(ctx, TRACE_OPEN);
}

int trace_fsync_return(struct pt_regs *ctx)
{
    return trace_return(ctx, TRACE_FSYNC);
}
"""
```

对hook点兼容性的处理，主要是`ext4_file_read_iter`与`generic_file_read_iter`（代码注释已说明）：

```PYTHON
# initialize BPF
b = BPF(text=bpf_text)

# Common file functions. See earlier comment about generic_file_read_iter().
if BPF.get_kprobe_functions(b'ext4_file_read_iter'):
    b.attach_kprobe(event="ext4_file_read_iter", fn_name="trace_read_entry")
else:
    b.attach_kprobe(event="generic_file_read_iter", fn_name="trace_read_entry")
b.attach_kprobe(event="ext4_file_write_iter", fn_name="trace_write_entry")
b.attach_kprobe(event="ext4_file_open", fn_name="trace_open_entry")
b.attach_kprobe(event="ext4_sync_file", fn_name="trace_fsync_entry")
if BPF.get_kprobe_functions(b'ext4_file_read_iter'):
    b.attach_kretprobe(event="ext4_file_read_iter", fn_name="trace_read_return")
else:
    b.attach_kretprobe(event="generic_file_read_iter", fn_name="trace_read_return")
b.attach_kretprobe(event="ext4_file_write_iter", fn_name="trace_write_return")
b.attach_kretprobe(event="ext4_file_open", fn_name="trace_open_return")
b.attach_kretprobe(event="ext4_sync_file", fn_name="trace_fsync_return")
```

##  0x07   代码：内存类


##  0x08 参考
-   [bpftrace一行教程](https://eunomia.dev/zh/tutorials/bpftrace-tutorial/)
-   [Linux探测工具BCC（可观测性）](https://www.cnblogs.com/charlieroro/p/13265252.html)
-   [BCC-文件系统组件分析](https://share.note.youdao.com/ynoteshare/index.html?id=18c1e114f98401ca3b2ececc67980726&type=note&_time=1760355931322)
-   [Linux eBPF BCC](https://www.yuanguohuo.com/2024/02/01/bcc-ebpf/)