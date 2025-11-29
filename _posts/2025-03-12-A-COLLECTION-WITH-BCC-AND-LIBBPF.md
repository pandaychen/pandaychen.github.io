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
-   [BCC](https://github.com/iovisor/bcc/tree/master/libbpf-tools)的实现
-   [](https://github.com/eunomia-bpf/bpf-developer-tutorial/tree/main/src)

##  0x01    基础


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

```CPP
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

```CPP
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

####    syncsnoop
[`syncsnoop`](https://github.com/iovisor/bcc/blob/master/libbpf-tools/syncsnoop.bpf.c)

##  0x08 参考
-   [BCC-文件系统组件分析](https://share.note.youdao.com/ynoteshare/index.html?id=18c1e114f98401ca3b2ececc67980726&type=note&_time=1760355931322)
