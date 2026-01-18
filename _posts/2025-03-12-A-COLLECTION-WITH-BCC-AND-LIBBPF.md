---
layout:     post
title:      BCC with libbpf
subtitle:   BCC all in one
date:       2025-02-22
author:     pandaychen
header-img:
catalog: true
tags:
    - eBPF
    - BCC
---

##  0x00    前言
测试机内核版本`5.4.119-1-tlinux4-0008`，python版本`3.6.8`

代码来源：
-   [BCC-tools](https://github.com/iovisor/bcc/tree/master/libbpf-tools)的实现
-   [eunomia-tools](https://github.com/eunomia-bpf/bpf-developer-tutorial/tree/main/src)

##  0x01    基础

####    内核态工具函数
1、`offsetof`

```cpp
//cc/libbpf/include/linux/kernel.h
#ifndef offsetof
#define offsetof(TYPE, MEMBER) ((size_t) &((TYPE *)0)->MEMBER)
#endif
```

2、`container_of`

```cpp
#ifndef container_of
#define container_of(ptr, type, member) ({                      \
        const typeof(((type *)0)->member) * __mptr = (ptr);     \
        (type *)((char *)__mptr - offsetof(type, member)); })
#endif
```

####    libbpf工具函数（重要）
1、`bpf_object__find_program_by_name`：根据给定的程序名称，从 BPF 对象中查找并返回相应的 eBPF 程序。如可以通过程序动态判断是否支持某个tracepoint/kprobe的挂载点，若不存在，需要使用这个API查找到对应的ebpf程序（struct bpf_program *），方便后续设置其`autoload`属性

```cpp
/*
- obj: 指向 BPF 对象的指针。该对象通常是通过加载 BPF ELF 文件生成的，包含多个 eBPF 程序。在libbpf-bootstrap框架中，可以通过skel->obj来获取
- name: 要查找的 eBPF 程序的名称。名称是一个字符串，应与 BPF 程序在源代码中的定义名称相匹配
- 返回值
成功时，返回指向找到的 eBPF 程序的指针（struct bpf_program *）
如果未找到匹配的程序，则失败，返回 NULL
*/
struct bpf_program * bpf_object__find_program_by_name(const struct bpf_object *obj, const char *name)
```

2、`bpf_program__set_autoload`：启用/禁用特定 eBPF 程序的自动加载。开发者可以决定某个 eBPF 程序在调用 `bpf_object__load` 时是否应被加载到内核中（实现动态加载ebpf hook）

```cpp
/*
- prog: 指向要设置自动加载属性的 eBPF 程序的指针（struct bpf_program *）
- autoload: 一个布尔值，指示是否应自动加载该程序。设为 true 表示启用自动加载，设为 false 表示禁用自动加载
- 返回值
成功时返回 0
失败时返回一个负值的错误码
*/
int bpf_program__set_autoload(struct bpf_program *prog, bool autoload)
```

```cpp
// 查找&&禁用
void disable_hook_autoload(struct bpf_object *obj, const char *prog_name) {
    struct bpf_program *prog = bpf_object__find_program_by_name(obj, prog_name);
    if (!prog) {
        fprintf(stderr, "Program %s not found in object\n", prog_name);
        return;
    }
    if (bpf_program__set_autoload(prog, false) != 0) {
        fprintf(stderr, "Failed to disable autoload for program %s\n", prog_name);
    } else {
        printf("Autoload disabled for program %s\n", prog_name);
    }

    return;
}
```

3、`ensure_core_btf`

4、`kprobe_exists`

5、`tracepoint_exists`

6、`fentry_can_attach`

TODO

在bcc的`libbpf-tools`实现中，能够看到大量使用此二者实现的用户态检测代码，主要用于kprobe/fentry等机制兼容性问题

```text
biolatency.c
biosnoop.c
biostacks.c
biotop.c
cachestat.c
cpudist.c
drsnoop.c
filelife.c
fsdist.c
fsslower.c
funclatency.c
hardirqs.c
klockstat.c
mdflush.c
memleak.c
mountsnoop.c
numamove.c
offcputime.c
opensnoop.c
readahead.c
runqlat.c
runqslower.c
sigsnoop.c
slabratetop.c
softirqs.c
solisten.c
statsnoop.c
tcpconnlat.c
tcppktlat.c
tcprtt.c
tcpsynbl.c
vfsstat.c
```

##  0x02    文件系统相关

####    filetop

```BASH
TID     COMM             READS  WRITES R_Kb    W_Kb    T FILE
1399240 sap1002          4      0      20479   0       R sockstat
3274711 sap1008          7      0      4900    0       R execve_info
3130102 clear            2      0      60      0       R xterm-256color
3130102 sh               2      0      8       0       R locale.alias
3130102 sh               14     0      5       0       R libc-2.28.so
3130102 sh               4      0      4       0       R cmdline
3130102 sh               4      0      4       0       R cmdline
3130100 filetop          2      0      2       0       R loadavg
3130102 sh               2      0      2       0       R cmdline
3130102 sh               2      0      2       0       R cmdline
3130102 sh               2      0      2       0       R cmdline
3130102 sh               6      0      1       0       R libtinfo.so.6.1
3130102 sh               2      0      1       0       R libdl-2.28.so
3130102 sh               2      0      1       0       R libonion_security.so.1.0.19
3130102 sh               2      0      1       0       R libonion_block_security.so.1.0.16
3130102 sh               2      0      1       0       R stat
3130102 sh               2      0      1       0       R stat
3130102 filetop          4      0      1       0       R ld-2.28.so
764743  sa1009          2      0      1       0       R stat
764737  sa1005          2      0      1       0       R stat
```

和python版本的类似，仅记录文件读写类型，读写失败时也将被记录。追踪`kprobe/vfs_read`、`kprobe/vfs_write`，记录文件操作类型、读写大小、文件名等信息

```cpp
SEC("kprobe/vfs_read")
int BPF_KPROBE(vfs_read_entry, struct file *file, char *buf, size_t count, loff_t *pos)
{
        return probe_entry(ctx, file, count, READ);
}

SEC("kprobe/vfs_write")
int BPF_KPROBE(vfs_write_entry, struct file *file, const char *buf, size_t count, loff_t *pos)
{
        return probe_entry(ctx, file, count, WRITE);
}
```

bpf hash表`entries`的key/value都是结构体：

```cpp
struct {
        __uint(type, BPF_MAP_TYPE_HASH);
        __uint(max_entries, MAX_ENTRIES);
        __type(key, struct file_id);
        __type(value, struct file_stat);
} entries SEC(".maps");

struct file_id {
        __u64 inode;
        __u32 dev;
        __u32 rdev;
        __u32 pid;
        __u32 tid;
};

struct file_stat {
        __u64 reads;
        __u64 read_bytes;
        __u64 writes;
        __u64 write_bytes;
        __u32 pid;
        __u32 tid;
        char filename[PATH_MAX];
        char comm[TASK_COMM_LEN];
        char type;
};
```

主要实现逻辑较为直观：

```cpp
// 获取dentry name
static void get_file_path(struct file *file, char *buf, size_t size)
{
        struct qstr dname;
        //先读dnanme，再读取dname.name
        dname = BPF_CORE_READ(file, f_path.dentry, d_name);
        bpf_probe_read_kernel(buf, size, dname.name);
}

static int probe_entry(struct pt_regs *ctx, struct file *file, size_t count, enum op op)
{
        __u64 pid_tgid = bpf_get_current_pid_tgid();
        __u32 pid = pid_tgid >> 32;
        __u32 tid = (__u32)pid_tgid;
        int mode;
        struct file_id key = {};
        struct file_stat *valuep;

        if (target_pid && target_pid != pid)
                return 0;

        mode = BPF_CORE_READ(file, f_inode, i_mode);
        if (regular_file_only && !S_ISREG(mode))
                return 0;

        key.dev = BPF_CORE_READ(file, f_inode, i_sb, s_dev);
        key.rdev = BPF_CORE_READ(file, f_inode, i_rdev);
        key.inode = BPF_CORE_READ(file, f_inode, i_ino);
        key.pid = pid;
        key.tid = tid;
        valuep = bpf_map_lookup_elem(&entries, &key);
        if (!valuep) {
                //初始化hash表
                bpf_map_update_elem(&entries, &key, &zero_value, BPF_ANY);
                valuep = bpf_map_lookup_elem(&entries, &key);
                if (!valuep)
                        return 0;
                valuep->pid = pid;
                valuep->tid = tid;
                bpf_get_current_comm(&valuep->comm, sizeof(valuep->comm));
                get_file_path(file, valuep->filename, sizeof(valuep->filename));
                if (S_ISREG(mode)) {
                        valuep->type = 'R';
                } else if (S_ISSOCK(mode)) {
                        valuep->type = 'S';
                } else {
                        valuep->type = 'O';
                }
        }
        if (op == READ) {
                valuep->reads++;
                valuep->read_bytes += count;
        } else {        /* op == WRITE */
                valuep->writes++;
                valuep->write_bytes += count;
        }
        return 0;
};
```

####    filelife
与python工具功能类似，该工具在`kprobe/vfs_create`、`kprobe/vfs_open`和`kprobe/security_inode_create`处追踪文件新建，其中`security_inode_create`（检查创建文件权限）作为兜底钩子，在文件创建时记录时间戳记录到map中；在`kprobe/vfs_unlink`删除文件的函数进入时，从map中读取文件创建/打开时的时间戳，计算时间差，收集文件路径等信息保存，在`kretprobe/vfs_unlink`时根据返回值是否是`0`判断删除文件是否成功，成功则将保存的信息发送给用户层

这里主要注意下libpbf(C)中是如何解决内核字段差异处理的，以`vfs_create`函数为例，在最近的Linux内核版本中，有三种典型的函数声明：

```cpp
int vfs_create(struct inode *dir, struct dentry *dentry, umode_t mode,
            bool want_excl);
int vfs_create(struct user_namespace *mnt_userns, struct inode *dir,
            struct dentry *dentry, umode_t mode, bool want_excl);
int vfs_create(struct mnt_idmap *idmap, struct inode *dir,
            struct dentry *dentry, umode_t mode, bool want_excl);
```

TODO

```cpp
SEC("kprobe/vfs_create")
int BPF_KPROBE(vfs_create, void *arg0, void *arg1, void *arg2)
{
        if (renamedata_has_old_mnt_userns_field()
                || renamedata_has_new_mnt_idmap_field())
                return probe_create(arg2);
        else
                return probe_create(arg1);
}

SEC("kprobe/vfs_open")
int BPF_KPROBE(vfs_open, struct path *path, struct file *file)
{
        struct dentry *dentry = BPF_CORE_READ(path, dentry);
        int fmode = BPF_CORE_READ(file, f_mode);

        if (!(fmode & FMODE_CREATED))
                return 0;

        return probe_create(dentry);
}

SEC("kprobe/security_inode_create")
int BPF_KPROBE(security_inode_create, struct inode *dir,
             struct dentry *dentry)
{
        return probe_create(dentry);
}

/**
 * In different kernel versions, function vfs_unlink() has two declarations,
 * and their parameter lists are as follows:
 *
 * int vfs_unlink(struct inode *dir, struct dentry *dentry,
 *        struct inode **delegated_inode);
 * int vfs_unlink(struct user_namespace *mnt_userns, struct inode *dir,
 *        struct dentry *dentry, struct inode **delegated_inode);
 * int vfs_unlink(struct mnt_idmap *idmap, struct inode *dir,
 *        struct dentry *dentry, struct inode **delegated_inode);
 */
SEC("kprobe/vfs_unlink")
int BPF_KPROBE(vfs_unlink, void *arg0, void *arg1, void *arg2)
{
        u64 id = bpf_get_current_pid_tgid();
        struct unlink_event unlink_event = {};
        struct create_arg *arg;
        u32 tgid = id >> 32;
        u32 tid = (u32)id;
        u64 delta_ns;
        bool has_arg = renamedata_has_old_mnt_userns_field()
                                || renamedata_has_new_mnt_idmap_field();

        arg = has_arg
                ? bpf_map_lookup_elem(&start, &arg2)
                : bpf_map_lookup_elem(&start, &arg1);
        if (!arg)
                return 0;   // missed entry

        delta_ns = bpf_ktime_get_ns() - arg->ts;

        unlink_event.delta_ns = delta_ns;
        unlink_event.tgid = tgid;
        unlink_event.dentry = has_arg ? arg2 : arg1;
        unlink_event.cwd_vfsmnt = arg->cwd_vfsmnt;

        bpf_map_update_elem(&currevent, &tid, &unlink_event, BPF_ANY);
        return 0;
}

SEC("kretprobe/vfs_unlink")
int BPF_KRETPROBE(vfs_unlink_ret)
{
        u64 id = bpf_get_current_pid_tgid();
        u32 tid = (u32)id;
        int ret = PT_REGS_RC(ctx);
        struct unlink_event *unlink_event;
        struct event *eventp;
        struct dentry *dentry;
        const u8 *qs_name_ptr;

        unlink_event = bpf_map_lookup_elem(&currevent, &tid);
        if (!unlink_event)
                return 0;
        bpf_map_delete_elem(&currevent, &tid);

        /* skip failed unlink */
        if (ret)
                return 0;

        eventp = reserve_buf(sizeof(*eventp));
        if (!eventp)
                return 0;

        eventp->tgid = unlink_event->tgid;
        eventp->delta_ns = unlink_event->delta_ns;
        bpf_get_current_comm(&eventp->task, sizeof(eventp->task));

        dentry = unlink_event->dentry;
        qs_name_ptr = BPF_CORE_READ(dentry, d_name.name);
        bpf_probe_read_kernel_str(&eventp->fname.pathes, sizeof(eventp->fname.pathes),
                           qs_name_ptr);
        eventp->fname.depth = 0;

        /* get full-path */
        if (full_path && eventp->fname.pathes[0] != '/')
                bpf_dentry_full_path(eventp->fname.pathes, NAME_MAX,
                                MAX_PATH_DEPTH,
                                unlink_event->dentry,
                                unlink_event->cwd_vfsmnt,
                                &eventp->fname.failed, &eventp->fname.depth);

        bpf_map_delete_elem(&start, &unlink_event->dentry);

        /* output */
        submit_buf(ctx, eventp, sizeof(*eventp));
        return 0;
}
```

####    mountsnoop
[`mountsnoop`](https://github.com/iovisor/bcc/blob/master/libbpf-tools/mountsnoop.bpf.c)，

####    syncsnoop
[`syncsnoop`](https://github.com/iovisor/bcc/blob/master/libbpf-tools/syncsnoop.bpf.c)

####    readahead：页表预读机制的细节（版本5.11.1）
readahead工具很有趣，其核心原理是追踪 Linux 内核页缓存 (Page Cache) 的预读 (Read-ahead) 机制，并计算预读页面的寿命与利用率。简单来说，它在记录如下数据：

1. 预读了多少页？
2. 这些页在多久之后被真正使用了？
3. 有多少页根本没被用到

内核态代码如下，涉及到如下hook点：

-  [`fentry/do_page_cache_ra`](https://elixir.bootlin.com/linux/v5.11.1/source/mm/readahead.c#L249)：
-  `fexit/__page_cache_alloc`：
-  `fexit/do_page_cache_ra`
-  `fentry/mark_page_accessed`

涉及核心hashtable的用途如下：

1. 核心数据结构：BPF Maps

-  `in_readahead`（Hash Map）： 存放当前正在执行预读任务的线程 PID
-  `birth`（Hash Map）：核心追踪表，key 是 `struct page *` 指针，value 是分配时的时间戳
-  `hist`：存放最终的统计结果，包括直方图数组 `slots`、总页面数 `total` 和未使用的页面数 `unused`

readahead工具的运行流程如下：

1、第一阶段：进入预读上下文 （The Context Switch），关联hook 点为`fentry/do_page_cache_ra`，当内核进入预读函数时，获取当前 PID，在 `in_readahead` hash表做一次记录，这里的目的是标记接下来的页面page分配动作属于预读行为

2、第二阶段：捕捉页面诞生的事件，hook 点为`fexit/__page_cache_alloc`（分配函数，当函数退出时触发），直接获取到函数的返回值 `ret`（即新分配的页面指针）。这里会检查当前 PID 是否在 `in_readahead` 中？如果该PID在hash表存在，说明这是个预读页，然后记录 `bpf_ktime_get_ns()` 到 `birth` 这个hash表中（key 为页面page指针），最后使用原子操作 `__sync_fetch_and_add` 增加全局计数器

3、第三阶段：退出预读上下文，关联hook 点为 `fexit/do_page_cache_ra`，当预读函数执行完毕退出时，从 `in_readahead` 中删除当前 PID。防止后续不相关的内存分配被计入

4、第四阶段：命中与统计，关联hook 点 `fentry/mark_page_accessed`，当内核通过 `mark_page_accessed` 访问一个页面时，先去 `birth` hash表查这个页面page指针，如果查找成功，说明预读页被命中（触发统计逻辑）

-  计算延迟：当前时间 - birth时间 = 页面在缓存中的生存时间
-  直方图统计：将延迟转换为 `log_2` 分度，存入直方图 `slots`
-  更新计数：既然page已经被标记使用了，那么`unused` 计数减 `1`
-  销毁记录：从 `birth` 表删除该页面page指针，这个page已经被使用了，所以需要从记录中清理掉

```cpp
//内核态实现
#define MAX_ENTRIES     10240

struct {
        __uint(type, BPF_MAP_TYPE_HASH);
        __uint(max_entries, MAX_ENTRIES);
        __type(key, u32);
        __type(value, u64);
} in_readahead SEC(".maps");

struct {
        __uint(type, BPF_MAP_TYPE_HASH);
        __uint(max_entries, MAX_ENTRIES);
        __type(key, struct page *);
        __type(value, u64);
} birth SEC(".maps");

struct hist hist = {};

SEC("fentry/do_page_cache_ra")
int BPF_PROG(do_page_cache_ra)
{
        u32 pid = bpf_get_current_pid_tgid();
        u64 one = 1;

        bpf_map_update_elem(&in_readahead, &pid, &one, 0);
        return 0;
}

static __always_inline
int alloc_done(struct page *page)
{
        u32 pid = bpf_get_current_pid_tgid();
        u64 ts;

        if (!bpf_map_lookup_elem(&in_readahead, &pid))
                return 0;

        ts = bpf_ktime_get_ns();
        bpf_map_update_elem(&birth, &page, &ts, 0);
        __sync_fetch_and_add(&hist.unused, 1);
        __sync_fetch_and_add(&hist.total, 1);

        return 0;
}

SEC("fexit/__page_cache_alloc")
int BPF_PROG(page_cache_alloc_ret, gfp_t gfp, struct page *ret)
{
        return alloc_done(ret);
}


SEC("fexit/do_page_cache_ra")
int BPF_PROG(do_page_cache_ra_ret)
{
        u32 pid = bpf_get_current_pid_tgid();

        bpf_map_delete_elem(&in_readahead, &pid);
        return 0;
}

static __always_inline
int mark_accessed(struct page *page)
{
        u64 *tsp, slot, ts = bpf_ktime_get_ns();
        s64 delta;

        tsp = bpf_map_lookup_elem(&birth, &page);
        if (!tsp)
                return 0;
        delta = (s64)(ts - *tsp);
        if (delta < 0)
                goto update_and_cleanup;
        slot = log2l(delta / 1000000U);
        if (slot >= MAX_SLOTS)
                slot = MAX_SLOTS - 1;
        __sync_fetch_and_add(&hist.slots[slot], 1);

update_and_cleanup:
        __sync_fetch_and_add(&hist.unused, -1);
        bpf_map_delete_elem(&birth, &page);

        return 0;
}

SEC("fentry/mark_page_accessed")
int BPF_PROG(mark_page_accessed, struct page *page)
{
        return mark_accessed(page);
}
```

这里有个小问题，为何可以使用page页面指针来作为hash表的key呢？

TODO
##      0x0    用户态代码的加载过程分析
TODO

##      0x0    内核的兼容性思考 

####    结构体成员变更的兼容性

在bcc基于libbpf实现功能的时候，采用了如下方式来解决兼容性的问题，参考[`core_fixes.bpf.h`](https://github.com/iovisor/bcc/blob/master/libbpf-tools/core_fixes.bpf.h)

```cpp
struct renamedata___x {
	struct user_namespace *old_mnt_userns;
	struct new_mnt_idmap *new_mnt_idmap;
} __attribute__((preserve_access_index));

static __always_inline bool renamedata_has_old_mnt_userns_field(void)
{
	if (bpf_core_field_exists(struct renamedata___x, old_mnt_userns))
		return true;
	return false;
}

static __always_inline bool renamedata_has_new_mnt_idmap_field(void)
{
	if (bpf_core_field_exists(struct renamedata___x, new_mnt_idmap))
		return true;
	return false;
}
```

####    tracepoint/kprobe：如何动态控制ebpf的钩子加载？
在不同版本的内核中，开发者需要检测hook的有效性。若相关的hook函数不存在，则可通过`bpf_object__find_program_by_name`查找到 eBPF 程序，再使用`bpf_program__set_autoload` 动态设置 eBPF 程序的`autoload`属性使其不加载

-       检查指定的 tracepoint 是否存在
-       检查指定的 kprobe 是否存在

1、如何检查tracepoint挂载点是否存在，通常检查如下两个路径`/sys/kernel/debug/tracing/events`、`/sys/kernel/tracing/events`是否存在以hook命名的目录

```cpp
bool tracepoint_exists(const char* tp_category, const char* tp_name)
{
    char path[256];
    snprintf(path, sizeof(path), "/sys/kernel/debug/tracing/events/%s/%s",
             tp_category, tp_name);
    auto ret = access(path, F_OK);
    if (ret == 0) {
        return true;
    }
    snprintf(path, sizeof(path), "/sys/kernel/tracing/events/%s/%s", tp_category, tp_name);
    ret = access(path, F_OK);
    if (ret == 0) {
        return true;
    }
    return false;
}
```

2、检查kprobe挂载点是否存在，kprobes允许动态挂载内核函数进行调试，通常检查如下路径：

-   `/sys/kernel/debug/tracing/available_filter_functions`
-   `/sys/kernel/tracing/available_filter_functions`

```cpp
bool kprobe_exists(const char *kprobe_name)
{
    FILE* file = fopen("/sys/kernel/debug/tracing/available_filter_functions", "r");
    if (!file) {
        file = fopen("/sys/kernel/tracing/available_filter_functions", "r");
        if (!file) {
            return false;
        }
    }
    char line[256];
    while (fgets(line, sizeof(line), file)) {
        line[strcspn(line, "\n")] = 0;
        if (strcmp(line, kprobe_name) == 0) {
            return true;
        }
    }
    return false;
}
```

####    动态设置 eBPF 程序的 autoload 属性


##  0x08 参考
-   [BCC-文件系统组件分析](https://share.note.youdao.com/ynoteshare/index.html?id=18c1e114f98401ca3b2ececc67980726&type=note&_time=1760355931322)
-   [libbpf-core_fixes](https://github.com/iovisor/bcc/blob/master/libbpf-tools/core_fixes.bpf.h)
