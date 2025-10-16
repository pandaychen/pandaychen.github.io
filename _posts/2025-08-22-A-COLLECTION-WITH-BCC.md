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

####    vfscount
用于统计一段时间内调用`vfs_*`相关函数的次数，通过`^vfs_.*`正则进行匹配，并进行排序，通过`cat /proc/kallsyms |grep " vfs"`可以获取到本机内核支持的`vfs_*`内核函数

```PYTHON
# load BPF program
b = BPF(text="""
#include <uapi/linux/ptrace.h>

struct key_t {
    u64 ip;
};

BPF_HASH(counts, struct key_t, u64, 256);

int do_count(struct pt_regs *ctx) {
    struct key_t key = {};
    key.ip = PT_REGS_IP(ctx);
    counts.atomic_increment(key);
    return 0;
}
""")
b.attach_kprobe(event_re="^vfs_.*", fn_name="do_count")
```

本机的采样结果如下：

```BASH
[root@VM-218-158-centos tools]# python3 vfscount.py 
Tracing... Ctrl-C to end.
^C
ADDR             FUNC                          COUNT
ffffffff81295351 b'vfs_fallocate'                  2
ffffffff812d5b51 b'vfs_fsync_range'               15
ffffffff812998e1 b'vfs_writev'                    22
ffffffff812c64a1 b'vfs_getxattr'                  52
ffffffff812a9cf1 b'vfs_mknod'                    109
ffffffff812a9a11 b'vfs_symlink'                  131
ffffffff812c76c1 b'vfs_setxattr'                 131
ffffffff812a94b1 b'vfs_rmdir'                    453
ffffffff812aa0b1 b'vfs_mkdir'                    459
ffffffff811f0361 b'vfs_fadvise'                  520
ffffffff812d8661 b'vfs_statfs'                   761
ffffffff812aa561 b'vfs_rename'                  1115
ffffffff812a9621 b'vfs_unlink'                  1159
ffffffff81310261 b'vfs_lock_file'              18439
ffffffff8129b151 b'vfs_write'                  76418
ffffffff8129ff51 b'vfs_statx_fd'              336288
ffffffff8129ffe1 b'vfs_statx'                 445290
ffffffff812a7a71 b'vfs_readlink'              503046
ffffffff81297041 b'vfs_open'                  634799
ffffffff8129fe91 b'vfs_getattr_nosec'         666308
ffffffff8129afa1 b'vfs_read'                  713822
```

一个小细节，以`vfs_read`为例，从ebpf程序中拿到的内核函数符号地址是`ffffffff8129afa1`，但是通过`cat /proc/kallsyms |grep " vfs_read"`获取到`vfs_read`的地址是`ffffffff8129afa0 T vfs_read`，两者不一致，思考下为什么？

TODO

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


##  0x06   代码：文件I/O类

####    basic：hook && 数据结构
hock位置
kprobe
security_inode_create

open/openat新建文件时不一定通过vfs_crete可能直接调用文件系统的create方法，但一定会在security_inode_create处检查创建文件权限。

linux-5.10.202/fs/namei.c: 3078

static struct dentry *lookup_open(struct nameidata *nd, struct file *file,      // 当前在namei层，根据建立文件的路径找到应该插入的位置
				  const struct open_flags *op,
				  bool got_write)
{
    /** open 查找文件位置 */

    /** 往下则未找到文件位置 */

    if (open_flag & O_CREAT) {                                                  // 可新建
        if (likely(got_write))
			create_error = may_o_create(&nd->path, dentry, mode);               // 在这里执行 security_inode_create

	error = dir_inode->i_op->create(dir_inode, dentry, mode,                    // 直接跨越vfs层调用文件系统的接口创建文件
						open_flag & O_EXCL);

    1
    2
    3
    4
    5
    6
    7
    8
    9
    10
    11
    12
    13
    14
    15
    16

vfs_*

用于对文件系统层的接口提供更上层的接口，通常由用户层程序发起的系统调用会通过vfs层的函数调用具体文件系统的方法。
但少部分时也有可能对文件的操作会跨过vfs层直接使用文件系统的接口，如果需要更精准的追踪需要追踪文件系统提供的接口。

    vfs_unlink 删除文件
    vfs_getattr 获取文件拓展属性，如文件的selinux标签、权能属性

lookup_fast

内核在处理字符串的文件路径，如"/home/kira/a"时，每一层级都会用一个struct dentry对象表示，如：层级"a"有一个对应的struct dentry对象，层级"a"的对象成员struct dentry *d_parent指向上一层"kira"的struct dentry对象的，“kira"的d_parent指向"home”，“home"的d_parent指向”/"根目录对象。这些对象在内核中会通过多种缓存机制减少频繁的申请释放以提升性能。文件位置查找时内核会通过一些函数先在缓存中查找。

lookup_fast函数仅在内核namei相关的地方使用到。namei的含义是从name转换到inode，也就是从路径字符串转换到内核中描述文件位置的结构体。lookup_fast函数的作用是将字符串表示路径的结构nd查找对应为inode结构。
具体查找的函数是__d_lookup_rcu和__d_lookup，两者的区别是__d_lookup_rcu性能开销更小。在内核VFS层的设计中，使用过的目录项对象有对应的hash表缓存，hash表中按照hash一致的为一个hash桶，在__d_loopup查找过程中，计算name的hash值，找到对应的hash桶，在这个hash桶内查找是否有缓存的struct dentry目录项对象。

linux-5.10.202/fs/namei.c: 1465

static struct dentry *lookup_fast(struct nameidata *nd,
				  struct inode **inode,
			          unsigned *seqp)
{
    if (nd->flags & LOOKUP_RCU) {
        dentry = __d_lookup_rcu(parent, &nd->last, &seq);
    } else {
        dentry = __d_lookup(parent, &nd->last);

linux-5.10.202/fs/dcache.c: 2343

/**
 * __d_lookup - search for a dentry (racy)
 * @parent: parent dentry
 * @name: qstr of name we wish to find
 * Returns: dentry, or NULL
 *
 * __d_lookup is like d_lookup, however it may (rarely) return a
 * false-negative result due to unrelated rename activity.
 *
 * __d_lookup is slightly faster by avoiding rename_lock read seqlock,
 * however it must be used carefully, eg. with a following d_lookup in
 * the case of failure.
 *
 * __d_lookup callers must be commented.
 */
struct dentry *__d_lookup(const struct dentry *parent, const struct qstr *name)
{
	unsigned int hash = name->hash;
	struct hlist_bl_head *b = d_hash(hash);     // 在此hash桶中查找是否有对应的目录项对象

    1
    2
    3
    4
    5
    6
    7
    8
    9
    10
    11
    12
    13
    14
    15
    16
    17
    18
    19
    20
    21
    22
    23
    24
    25
    26
    27
    28
    29
    30
    31
    32

d_lookup

d_lookup函数的作用是在目录parent中查找文件名为name的文件。d_lookup只会调用__d_lookup，有一定的锁开销，对应dcstat结果中对应SLOW条目

linux-5.10.202/fs/dcache.c: 2317

/**
 * d_lookup - search for a dentry
 * @parent: parent dentry
 * @name: qstr of name we wish to find
 * Returns: dentry, or NULL
 *
 * d_lookup searches the children of the parent dentry for the name in
 * question. If the dentry is found its reference count is incremented and the
 * dentry is returned. The caller must use dput to free the entry when it has
 * finished using it. %NULL is returned if the dentry does not exist.
 */
struct dentry *d_lookup(const struct dentry *parent, const struct qstr *name) {
    dentry = __d_lookup(parent, name);

    1
    2
    3
    4
    5
    6
    7
    8
    9
    10
    11
    12
    13
    14
    15

namei层在查找路径位置时通常会先尝试使用RCU小开锁的形式查找缓存，找不到则使用普通锁方式查找缓存，两个缓存再找不到就需要读取硬盘设备查找了。

linux-5.10.202/fs/namei.c: 2364

int filename_lookup(int dfd, struct filename *name, unsigned flags,
		    struct path *path, struct path *root)
{
    retval = path_lookupat(&nd, flags | LOOKUP_RCU, path);
	if (unlikely(retval == -ECHILD))
		retval = path_lookupat(&nd, flags, path);
	if (unlikely(retval == -ESTALE))
		retval = path_lookupat(&nd, flags | LOOKUP_REVAL, path);

    1
    2
    3
    4
    5
    6
    7
    8
    9
    10

有两种锁和文件可能同时存在读和写等其他状态有关。
数据结构
inode

inode代表文件的元数据、属性。如文件的创建时间、修改时间、大小，但linux内核中也会在此数据结构中添加对inode指向文件操作的接口，接口由文件系统层提供。

linux-5.10.202/include/linux/fs.h：610

struct inode {
    /** 属性 */
	kuid_t			i_uid;
	kgid_t			i_gid;
	
	/** 文件系统提供的操作接口 */
	const struct inode_operations	*i_op;

    1
    2
    3
    4
    5
    6
    7
    8
    9

dentry

用于追溯文件路径的结构体，在eBPF程序中通常使用该结构体解析文件路径。

linux-5.10.202/include/linux/dcache.h: 89
struct dentry {
    struct dentry *d_parent;	/* parent directory 指向父层级 父层级的d_name是父层级文件夹的名称 */
    struct qstr d_name;         // 文件名，当前层级

    1
    2
    3
    4

ps: 仅仅使用dentry不能完整取出文件的绝对路径，文件路径中可能存在挂载点，需要挂载点结构体共同配合才能解析此类文件完整绝对路径。
————————————————
版权声明：本文为CSDN博主「Kira Skyler」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。
原文链接：https://blog.csdn.net/weixin_42544902/article/details/146496222


####    vfsstat：操作计数
统计一段时间时间内 VFS 调用的次数，与前文`vfscount`不同，该工具不会输出`comm`，包括如下状态：

-   `READ`
-   `WRITE`
-   `FSYNC`
-   `OPEN`
-   `CREATE`
-   `UNLINK`
-   `MKDIR`
-   `RMDIR`

采样输出如下：

```BASH
[root@VM-X-X-centos tools]# python3 vfsstat.py 
TIME         READ/s  WRITE/s  FSYNC/s   OPEN/s CREATE/s UNLINK/s  MKDIR/s  RMDIR/s
13:24:23:       128       54        0       38        0        0        0        0
13:24:24:       100       41        0       29        0        0        0        0
13:24:25:       318       38        0      240        0        0        0        0
13:24:26:        87       33        0       25        0        0        0        0
```

内核态代码如下，这里需要关注下对kfunc机制（`>=5.15`内核）的使用，kfunc (Kernel Functions) 机制是Linux 内核中一些被显式导出供 eBPF 程序调用的函数。与传统 helper 函数的区别是，在 kfunc 出现之前，eBPF 程序只能调用一组固定的、预定义好的 bpf_helper函数（如 `bpf_map_lookup_elem`）。kfunc 机制则灵活得多，允许内核开发者将更多内核函数暴露给 eBPF，而无需修改核心的 eBPF 子系统

笔者的内核版本是`5.4.119-1-tlinux4-0008`，不支持kfunc，使用的还是`vfs_*`系列的kprobe钩子，在BCC中是通过`support_kfunc`[方法](https://github.com/iovisor/bcc/blob/master/src/python/bcc/__init__.py#L1142)检查是否支持

```PYTHON
def support_kfunc():
        # there's no trampoline support for other than x86_64 arch
        if platform.machine() != 'x86_64':
            return False
        if not lib.bpf_has_kernel_btf():
            return False
        # kernel symbol "bpf_trampoline_link_prog" indicates kfunc support
        if BPF.ksymname("bpf_trampoline_link_prog") != -1:
            return True
        return False
```

内核态代码：

```PYTHON
# load BPF program
bpf_text = """

enum stat_types {
    S_READ = 1,
    S_WRITE,
    S_FSYNC,
    S_OPEN,
    S_CREATE,
    S_UNLINK,
    S_MKDIR,
    S_RMDIR,
    S_MAXSTAT
};

BPF_ARRAY(stats, u64, S_MAXSTAT);

static void stats_try_increment(int key) {
    PID_FILTER
    stats.atomic_increment(key);
}
"""

bpf_text_kprobe = """
void do_read(struct pt_regs *ctx) { stats_try_increment(S_READ); }
void do_write(struct pt_regs *ctx) { stats_try_increment(S_WRITE); }
void do_fsync(struct pt_regs *ctx) { stats_try_increment(S_FSYNC); }
void do_open(struct pt_regs *ctx) { stats_try_increment(S_OPEN); }
void do_create(struct pt_regs *ctx) { stats_try_increment(S_CREATE); }
void do_unlink(struct pt_regs *ctx) { stats_try_increment(S_UNLINK); }
void do_mkdir(struct pt_regs *ctx) { stats_try_increment(S_MKDIR); }
void do_rmdir(struct pt_regs *ctx) { stats_try_increment(S_RMDIR); }
"""

bpf_text_kfunc = """
KFUNC_PROBE(vfs_read)         { stats_try_increment(S_READ); return 0; }
KFUNC_PROBE(vfs_write)        { stats_try_increment(S_WRITE); return 0; }
KFUNC_PROBE(vfs_fsync_range)  { stats_try_increment(S_FSYNC); return 0; }
KFUNC_PROBE(vfs_open)         { stats_try_increment(S_OPEN); return 0; }
KFUNC_PROBE(vfs_create)       { stats_try_increment(S_CREATE); return 0; }
KFUNC_PROBE(vfs_unlink)       { stats_try_increment(S_UNLINK); return 0; }
KFUNC_PROBE(vfs_mkdir)        { stats_try_increment(S_MKDIR); return 0; }
KFUNC_PROBE(vfs_rmdir)        { stats_try_increment(S_RMDIR); return 0; }
"""

is_support_kfunc = BPF.support_kfunc()
if is_support_kfunc:
    bpf_text += bpf_text_kfunc
else:
    bpf_text += bpf_text_kprobe

if args.pid:
    bpf_text = bpf_text.replace('PID_FILTER', """
    u32 pid = bpf_get_current_pid_tgid() >> 32;
    if (pid != %s) {
        return;
    }
    """ % args.pid)
else:
    bpf_text = bpf_text.replace('PID_FILTER', '')

if debug or args.ebpf:
    print(bpf_text)
    if args.ebpf:
        exit()

b = BPF(text=bpf_text)
if not is_support_kfunc:
    b.attach_kprobe(event="vfs_read",         fn_name="do_read")
    b.attach_kprobe(event="vfs_write",        fn_name="do_write")
    b.attach_kprobe(event="vfs_fsync_range",  fn_name="do_fsync")
    b.attach_kprobe(event="vfs_open",         fn_name="do_open")
    b.attach_kprobe(event="vfs_create",       fn_name="do_create")
    b.attach_kprobe(event="vfs_unlink",       fn_name="do_unlink")
    b.attach_kprobe(event="vfs_mkdir",        fn_name="do_mkdir")
    b.attach_kprobe(event="vfs_rmdir",        fn_name="do_rmdir")

```


####    filelife（文件存活时长跟踪）
本工具通过跟踪`vsf_create/vfs_open/security_inode_create/vfs_unlink`内核函数，来检测文件从创建到删除的存活时间（在`filelife`运行期间创建/打开后又被删除的文件），可以观测到哪些线程在频繁创建和删除文件，需要处理上述函数在内核版本的兼容性，思考下，为何对于文件创建要选择上述三个hook点？根据前文的分析，可以了解创建文件有两种方式，第一种是通过系统调用如`creat`创建文件，第二种是在`open/openat`中指定参数，打开不存在文件时进行创建

```CPP

```

`filelife`在`kprobe/vfs_create`、`kprobe/vfs_open`和`kprobe/security_inode_create`处追踪文件新建事件（可能会有重复），注意 **`open/openat`新建文件（参考[前文]()对open的内核源码分析）时不一定通过`vfs_create`可能直接调用文件系统的`create`方法，但一定会在`security_inode_create`处检查创建文件权限**，在文件创建时记录时间戳记录到map中，最后在`kprobe/vfs_unlink`删除文件的函数进入时刻，从map中读取文件创建/打开时的时间戳，计算时间差，收集文件路径等信息保存，在`kretprobe/vfs_unlink`时根据返回值是否是`0`判断删除文件是否成功

```PYTHON
# define BPF program
bpf_text = """
......

struct data_t {
    u32 pid;
    u64 delta;
    char comm[TASK_COMM_LEN];
    char fname[DNAME_INLINE_LEN];
    /* private */
    void *dentry;
};

BPF_HASH(birth, struct dentry *);
BPF_HASH(unlink_data, u32, struct data_t);
BPF_PERF_OUTPUT(events);

static int probe_dentry(struct pt_regs *ctx, struct dentry *dentry)
{
    u32 pid = bpf_get_current_pid_tgid() >> 32;
    FILTER

    u64 ts = bpf_ktime_get_ns();
    birth.update(&dentry, &ts);

    return 0;
}

// trace file creation time
TRACE_CREATE_FUNC
{
    return probe_dentry(ctx, dentry);
};

// trace file security_inode_create time
int trace_security_inode_create(struct pt_regs *ctx, struct inode *dir,
        struct dentry *dentry)
{
    return probe_dentry(ctx, dentry);
};

// trace file open time
int trace_open(struct pt_regs *ctx, struct path *path, struct file *file)
{
    struct dentry *dentry = path->dentry;

    //过滤掉非FMODE_CREATED的操作
    if (!(file->f_mode & FMODE_CREATED)) {
        return 0;
    }

    return probe_dentry(ctx, dentry);
};

// trace file deletion and output details
TRACE_UNLINK_FUNC
{
    struct data_t data = {};
    u64 pid_tgid = bpf_get_current_pid_tgid();
    u32 pid = pid_tgid >> 32;
    u32 tid = (u32)pid_tgid;

    FILTER

    u64 *tsp, delta;
    tsp = birth.lookup(&dentry);
    if (tsp == 0) {
        return 0;   // missed create
    }

    delta = (bpf_ktime_get_ns() - *tsp) / 1000000;

    struct qstr d_name = dentry->d_name;
    if (d_name.len == 0)
        return 0;

    if (bpf_get_current_comm(&data.comm, sizeof(data.comm)) == 0) {
        data.pid = pid;
        data.delta = delta;
        bpf_probe_read_kernel(&data.fname, sizeof(data.fname), d_name.name);
    }

    /* record dentry, only delete from birth if unlink successful */
    data.dentry = dentry;

    unlink_data.update(&tid, &data);
    return 0;
}

int trace_unlink_ret(struct pt_regs *ctx)
{
    int ret = PT_REGS_RC(ctx);
    struct data_t *data;
    u32 tid = (u32)bpf_get_current_pid_tgid();

    data = unlink_data.lookup(&tid);
    if (!data)
        return 0;

    /* delete it any way */
    unlink_data.delete(&tid);

    /* Skip failed unlink */
    if (ret)
        return 0;

    birth.delete((struct dentry **)&data->dentry);
    events.perf_submit(ctx, data, sizeof(*data));

    return 0;
}
"""

trace_create_text_1="""
int trace_create(struct pt_regs *ctx, struct inode *dir, struct dentry *dentry)
"""
trace_create_text_2="""
int trace_create(struct pt_regs *ctx, struct user_namespace *mnt_userns,
        struct inode *dir, struct dentry *dentry)
"""
trace_create_text_3="""
int trace_create(struct pt_regs *ctx, struct mnt_idmap *idmap,
        struct inode *dir, struct dentry *dentry)
"""

trace_unlink_text_1="""
int trace_unlink(struct pt_regs *ctx, struct inode *dir, struct dentry *dentry)
"""
trace_unlink_text_2="""
int trace_unlink(struct pt_regs *ctx, struct user_namespace *mnt_userns,
        struct inode *dir, struct dentry *dentry)
"""
trace_unlink_text_3="""
int trace_unlink(struct pt_regs *ctx, struct mnt_idmap *idmap,
        struct inode *dir, struct dentry *dentry)
"""

if args.pid:
    bpf_text = bpf_text.replace('FILTER',
        'if (pid != %s) { return 0; }' % args.pid)
else:
    bpf_text = bpf_text.replace('FILTER', '')
if debug or args.ebpf:
    print(bpf_text)
    if args.ebpf:
        exit()

if BPF.kernel_struct_has_field(b'renamedata', b'new_mnt_idmap') == 1:
    bpf_text = bpf_text.replace('TRACE_CREATE_FUNC', trace_create_text_3)
    bpf_text = bpf_text.replace('TRACE_UNLINK_FUNC', trace_unlink_text_3)
elif BPF.kernel_struct_has_field(b'renamedata', b'old_mnt_userns') == 1:
    bpf_text = bpf_text.replace('TRACE_CREATE_FUNC', trace_create_text_2)
    bpf_text = bpf_text.replace('TRACE_UNLINK_FUNC', trace_unlink_text_2)
else:
    bpf_text = bpf_text.replace('TRACE_CREATE_FUNC', trace_create_text_1)
    bpf_text = bpf_text.replace('TRACE_UNLINK_FUNC', trace_unlink_text_1)

# initialize BPF
b = BPF(text=bpf_text)
b.attach_kprobe(event="vfs_create", fn_name="trace_create")
# newer kernels may don't fire vfs_create, call vfs_open instead:
b.attach_kprobe(event="vfs_open", fn_name="trace_open")
# newer kernels (say, 4.8) may don't fire vfs_create, so record (or overwrite)
# the timestamp in security_inode_create():
if BPF.get_kprobe_functions(b"security_inode_create"):
    b.attach_kprobe(event="security_inode_create", fn_name="trace_security_inode_create")
b.attach_kprobe(event="vfs_unlink", fn_name="trace_unlink")
b.attach_kretprobe(event="vfs_unlink", fn_name="trace_unlink_ret")
```

输出如下：
```BASH
[root@VM-X-X-centos ~]# touch testfile;sleep 5;rm testfile

[root@VM-X-X-centos tools]# python3 filelife.py 
TIME     PID     COMM             AGE(s)  FILE
10:42:36 1432827 rm               5.01    testfile
```

####    fileslower：检测读写耗时长的文件
运行结果如下：
```BASH
[root@VM-X-X-centos tools]# python3 fileslower.py 10
Tracing sync read/writes slower than 10 ms
TIME(s)  COMM           TID    D BYTES   LAT(ms) FILENAME
26.720   bash           1432366 W 4         22.76 tracee-x86_64.v0.21.0.tar.gz
```

相关内核态代码如下，关联hook为`__vfs_read[vfs_read]/__vfs_write[vfs_write]`，注意其中的这么一句话`skip non-sync I/O; see kernel code for __vfs_read()`（写同样）

```PYTHON
# define BPF program
bpf_text = """
......

enum trace_mode {
    MODE_READ,
    MODE_WRITE
};

struct val_t {
    u32 sz;
    u64 ts;
    u32 name_len;
    // de->d_name.name may point to de->d_iname so limit len accordingly
    char name[DNAME_INLINE_LEN];
    char comm[TASK_COMM_LEN];
};

struct data_t {
    enum trace_mode mode;
    u32 pid;
    u32 sz;
    u64 delta_us;
    u32 name_len;
    char name[DNAME_INLINE_LEN];
    char comm[TASK_COMM_LEN];
};

BPF_HASH(entryinfo, pid_t, struct val_t);
BPF_PERF_OUTPUT(events);

// store timestamp and size on entry
static int trace_rw_entry(struct pt_regs *ctx, struct file *file,
    char __user *buf, size_t count)
{
    u32 tgid = bpf_get_current_pid_tgid() >> 32;
    if (TGID_FILTER)
        return 0;

    u32 pid = bpf_get_current_pid_tgid();

    // skip I/O lacking a filename
    struct dentry *de = file->f_path.dentry;
    int mode = file->f_inode->i_mode;
    if (de->d_name.len == 0 || TYPE_FILTER)
        return 0;

    // store size and timestamp by pid
    struct val_t val = {};
    val.sz = count;
    val.ts = bpf_ktime_get_ns();

    struct qstr d_name = de->d_name;
    val.name_len = d_name.len;
    bpf_probe_read_kernel(&val.name, sizeof(val.name), d_name.name);
    bpf_get_current_comm(&val.comm, sizeof(val.comm));
    entryinfo.update(&pid, &val);

    return 0;
}

int trace_read_entry(struct pt_regs *ctx, struct file *file,
    char __user *buf, size_t count)
{
    // skip non-sync I/O; see kernel code for __vfs_read()
    if (!(file->f_op->read_iter))
        return 0;
    return trace_rw_entry(ctx, file, buf, count);
}

int trace_write_entry(struct pt_regs *ctx, struct file *file,
    char __user *buf, size_t count)
{
    // skip non-sync I/O; see kernel code for __vfs_write()
    if (!(file->f_op->write_iter))
        return 0;
    return trace_rw_entry(ctx, file, buf, count);
}

// output
static int trace_rw_return(struct pt_regs *ctx, int type)
{
    struct val_t *valp;
    u32 pid = bpf_get_current_pid_tgid();

    valp = entryinfo.lookup(&pid);
    if (valp == 0) {
        // missed tracing issue or filtered
        return 0;
    }
    u64 delta_us = (bpf_ktime_get_ns() - valp->ts) / 1000;
    entryinfo.delete(&pid);
    if (delta_us < MIN_US)
        return 0;

    struct data_t data = {};
    data.mode = type;
    data.pid = pid;
    data.sz = valp->sz;
    data.delta_us = delta_us;
    data.name_len = valp->name_len;
    bpf_probe_read_kernel(&data.name, sizeof(data.name), valp->name);
    bpf_probe_read_kernel(&data.comm, sizeof(data.comm), valp->comm);
    events.perf_submit(ctx, &data, sizeof(data));

    return 0;
}

int trace_read_return(struct pt_regs *ctx)
{
    return trace_rw_return(ctx, MODE_READ);
}

int trace_write_return(struct pt_regs *ctx)
{
    return trace_rw_return(ctx, MODE_WRITE);
}

"""
bpf_text = bpf_text.replace('MIN_US', str(min_ms * 1000))
if args.tgid:
    bpf_text = bpf_text.replace('TGID_FILTER', 'tgid != %d' % tgid)
else:
    bpf_text = bpf_text.replace('TGID_FILTER', '0')
if args.all_files:
    bpf_text = bpf_text.replace('TYPE_FILTER', '0')
else:
    bpf_text = bpf_text.replace('TYPE_FILTER', '!S_ISREG(mode)')

if debug or args.ebpf:
    print(bpf_text)
    if args.ebpf:
        exit()

# initialize BPF
b = BPF(text=bpf_text)

# I'd rather trace these via new_sync_read/new_sync_write (which used to be
# do_sync_read/do_sync_write), but those became static. So trace these from
# the parent functions, at the cost of more overhead, instead.
# Ultimately, we should be using [V]FS tracepoints.
try:
    b.attach_kprobe(event="__vfs_read", fn_name="trace_read_entry")
    b.attach_kretprobe(event="__vfs_read", fn_name="trace_read_return")
except Exception:
    print('Current kernel does not have __vfs_read, try vfs_read instead')
    b.attach_kprobe(event="vfs_read", fn_name="trace_read_entry")
    b.attach_kretprobe(event="vfs_read", fn_name="trace_read_return")
try:
    b.attach_kprobe(event="__vfs_write", fn_name="trace_write_entry")
    b.attach_kretprobe(event="__vfs_write", fn_name="trace_write_return")
except Exception:
    print('Current kernel does not have __vfs_write, try vfs_write instead')
    b.attach_kprobe(event="vfs_write", fn_name="trace_write_entry")
    b.attach_kretprobe(event="vfs_write", fn_name="trace_write_return")
```

####    ext4slower
ext4slower：[`ext4slower`](https://github.com/iovisor/bcc/blob/v0.35.0/tools/ext4slower.py)会跟踪ext4文件系统的读/写/打开和`fsyncs`等操作，并在超过阈值后打印信息，借此排查慢速IO问题，该工具可以在 VFS->文件系统接口层面进行跟踪，内核态代码如下：

针对四种操作类型`R/W/O/S`的监控路径，比较单一，仅仅是基于kprobe/kretprobe的耗时，四种类型之间没有计算关联

-   `kprobe(xxx)`：在 xxx开始时记录时间戳
-   `kretprobe(xxx)`：在 xxx结束时再次获取时间，计算出耗时，并且能够访问到xxx内核函数的返回值

由于通常的磁盘I/O处理是异步的，因此不太适合在VFS实现这个需求，所以比较适合在文件系统层次分析慢速磁盘I/O

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

####    xfsslower
原理同`ext4slower`，统计一段时间内存 XFS 文件系统中文件的读写时间

-   `T`： 文件类型：R(常规文件)，S(Socket)，O(other)
-    `OFF_KB`：File offset

####    biolatency

####    filegone：追踪文件重命名/删除
实现机制是在`vfs_rename`中追踪文件重命名操作，在`vfs_unlink`中追踪文件删除操作，兼顾了内核兼容性

```BASH
[root@VM-X-X-centos tools]# python3 filegone.py
TIME     PID     COMM             ACTION FILE
18:22:49 702124 in:imjournal     82 imjournal.state.tmp > imjournal.state
18:22:58 1665296 bash             68 sh-thd.zsIEk6
18:22:58 1665337 rm               68 ttttt
```

####    filetop：文件读写数据量TOP
该工具用于统计一段时间内读写较高的文件，主要是基于`vfs_read/vfs_write`做统计，输出格式说明：

-   `READS/WRITES`：读写次数
-   `R_Kb/W_Kb`：读写数据量
-   `T`：文件类型，R（常规文件），S（Socket），O（other）

采样输出如下：

```BASH
16:15:30 loadavg: 0.16 0.10 0.06 1/379 1579177

TID     COMM             READS  WRITES R_Kb    W_Kb    T FILE
764728  sap1002          12     0      61439   0       R sockstat
3274711 sap1008          15     0      10500   0       R execve_info
764730  sap1003          12     0      95      0       R cmdline
764730  sap1003          12     0      95      0       R cmdline
1579177 python3          10     0      90      0       R cmdline
```

内核态代码如下，关注下面这几个成员的获取及判断：

-   inode：`file->f_inode->i_ino`
-   dev：`file->f_inode->i_sb->s_dev`
-   rdev：`file->f_inode->i_rdev`
-   mode：`file->f_inode->i_mode`
-   ：`file->f_path.dentry->d_parent->d_inode->i_ino`

```PYTHON
# define BPF program
bpf_text = """
#include <uapi/linux/ptrace.h>
#include <linux/blkdev.h>

enum { __BCC_DNAME_INLINE_LEN = DNAME_INLINE_LEN };
#undef DNAME_INLINE_LEN
#define DNAME_INLINE_LEN __BCC_DNAME_INLINE_LEN

// the key for the output summary
struct info_t {
    unsigned long inode;
    dev_t dev;
    dev_t rdev;
    u32 pid;
    u32 name_len;
    char comm[TASK_COMM_LEN];
    // de->d_name.name may point to de->d_iname so limit len accordingly
    char name[DNAME_INLINE_LEN];
    char type;
};

// the value of the output summary
struct val_t {
    u64 reads;
    u64 writes;
    u64 rbytes;
    u64 wbytes;
};

BPF_HASH(counts, struct info_t, struct val_t);

// vfs_read/vfs_write的参数
static int do_entry(struct pt_regs *ctx, struct file *file,
    char __user *buf, size_t count, int is_read)
{
    u32 tgid = bpf_get_current_pid_tgid() >> 32;
    if (TGID_FILTER)
        return 0;

    u32 pid = bpf_get_current_pid_tgid();

    // skip I/O lacking a filename
    struct dentry *de = file->f_path.dentry;
    int mode = file->f_inode->i_mode;
    struct qstr d_name = de->d_name;   //获取dentry的qstr
    if (d_name.len == 0 || TYPE_FILTER)
        return 0;

    // skip if not in the specified directory
    if (DIRECTORY_FILTER)
        return 0;

    // store counts and sizes by pid & file
    struct info_t info = {
        .pid = pid,
        .inode = file->f_inode->i_ino,
        .dev = file->f_inode->i_sb->s_dev,
        .rdev = file->f_inode->i_rdev,
    };
    bpf_get_current_comm(&info.comm, sizeof(info.comm));
    info.name_len = d_name.len;
    bpf_probe_read_kernel(&info.name, sizeof(info.name), d_name.name);
    if (S_ISREG(mode)) {
        info.type = 'R';
    } else if (S_ISSOCK(mode)) {
        info.type = 'S';
    } else {
        info.type = 'O';
    }

    struct val_t *valp, zero = {};
    valp = counts.lookup_or_try_init(&info, &zero);
    if (valp) {
        if (is_read) {
            valp->reads++;
            // 累加读数据
            valp->rbytes += count;
        } else {
            valp->writes++;
            // 累加写数据
            valp->wbytes += count;
        }
    }

    return 0;
}

int trace_read_entry(struct pt_regs *ctx, struct file *file,
    char __user *buf, size_t count)
{
    return do_entry(ctx, file, buf, count, 1);
}

int trace_write_entry(struct pt_regs *ctx, struct file *file,
    char __user *buf, size_t count)
{
    return do_entry(ctx, file, buf, count, 0);
}
"""

......

if args.directory:
    try:
        directory_inode = os.lstat(args.directory)[stat.ST_INO]
        print(f'Tracing directory: {args.directory} (Inode: {directory_inode})')
        bpf_text = bpf_text.replace('DIRECTORY_FILTER',  'file->f_path.dentry->d_parent->d_inode->i_ino != %d' % directory_inode)
    except (FileNotFoundError, PermissionError) as e:
        print(f'Error accessing directory {args.directory}: {e}')
        exit(1)


# initialize BPF
b = BPF(text=bpf_text)
if args.read_only and args.write_only:
    raise Exception("Both read-only and write-only flags passed")
elif args.read_only:
    b.attach_kprobe(event="vfs_read", fn_name="trace_read_entry")
elif args.write_only:
    b.attach_kprobe(event="vfs_write", fn_name="trace_write_entry")
else:
    b.attach_kprobe(event="vfs_read", fn_name="trace_read_entry")
    b.attach_kprobe(event="vfs_write", fn_name="trace_write_entry")
```

####    biosnoop
[`biosnoop`](https://github.com/iovisor/bcc/blob/v0.35.0/tools/biosnoop.py)跟踪每个块设备的I/O，并打印每个I/O的延迟，使用时可以结合`biolatency`工具，首先使用`python3 biolatency.py -D`找出延迟（分布）大的磁盘，然后使用`biosnoop`找出导致延迟的进程

##  0x07   代码：内存类


##  0x08 参考
-   [bpftrace一行教程](https://eunomia.dev/zh/tutorials/bpftrace-tutorial/)
-   [Linux探测工具BCC（可观测性）](https://www.cnblogs.com/charlieroro/p/13265252.html)
-   [BCC-文件系统组件分析](https://share.note.youdao.com/ynoteshare/index.html?id=18c1e114f98401ca3b2ececc67980726&type=note&_time=1760355931322)
-   [Linux eBPF BCC](https://www.yuanguohuo.com/2024/02/01/bcc-ebpf/)