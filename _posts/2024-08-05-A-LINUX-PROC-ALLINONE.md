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
`proc`文件系统（[procfs](https://www.kernel.org/doc/Documentation/filesystems/proc.txt)）是一种基于内核的VFS，以文件系统目录和文件形式，提供一个指向内核数据结构的接口，通过它能够查看和改变各种系统属性。从开发者角度说，Procfs 是一种特殊的虚拟文件系统，可以挂载到用户的目录树中，允许用户空间中的进程使用系统调用（如`write`/`read`等）方便地读取内核信息

本文内核代码基于 [v4.11.6](https://elixir.bootlin.com/linux/v4.11.6/source/include) 版本

![vfs](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/vfs/procs/vfs.jpg)

-   procfs 它不依赖物理存储设备，而是由内核动态生成文件和目录，用于暴露系统信息（如进程状态、硬件配置、内核参数等），它通过注册到 VFS 的机制，将自己挂载到文件系统树（`/proc`），并遵循 VFS 的接口规范（如`inode_operations`、`file_operations`）
-   内核数据结构：procfs 的内容（如 `/proc/cpuinfo`、`/proc/meminfo`等）由内核动态生成，通过回调函数响应读/写操作
-   VFS 接口：procfs 需要实现 VFS 定义的操作函数（例如 `open()`、`read()`、`readdir()`），才能被用户空间访问
-   procfs 的功能和内容完全由内核管理，通过 `mount -t proc proc /proc` 命令将 procfs 挂载到 VFS 树中，使其成为文件系统的一部分

####    挂载proc

```BASH
# Mount the `proc` device under `/proc`
# with the filesystem type specified as
# `proc`.
#
# note.: you could mount procfs pretty much
#        anywhere you want and specify any
#        device name.
#
#
#         .------------> fs type
#         |    
#         |     .------> device (dummy identifier
#         |    |         in this case - anything)
#         |    |
#         |    |     .-> location
#         |    |     |
#         |    |     |
mount -t proc proc /proc
```

##  0x01    引言：访问/proc文件
`/proc/`目录下包含了文件和目录（pid维度）

```BASH
[root@VM-X-X-tencentos ~]# ls /proc/ -al
total 4
dr-xr-xr-x 270 root  root               0 Nov  7 07:06 .
dr-xr-xr-x  19 root  root            4096 Nov  7 16:46 ..
dr-xr-xr-x   8 root  root               0 Nov  7 07:06 1
......
dr-xr-xr-x   8 root  root               0 Nov  7 07:06 15
dr-xr-xr-x   8 root  root               0 Nov  7 07:07 996
dr-xr-xr-x   3 root  root               0 Nov  7 07:06 acpi
-r--r--r--   1 root  root               0 Nov  7 07:07 bt_stat
-r--r--r--   1 root  root               0 Nov  7 16:45 buddyinfo
dr-xr-xr-x   4 root  root               0 Nov  7 07:07 bus
-r--r--r--   1 root  root               0 Nov  7 07:06 cgroups
-r--r--r--   1 root  root               0 Nov  7 07:06 cmdline
-r--r--r--   1 root  root           30094 Nov  7 07:07 config.gz
-r--r--r--   1 root  root               0 Nov  7 16:45 consoles
-r--r--r--   1 root  root               0 Nov  7 07:06 cpuinfo
-r--r--r--   1 root  root               0 Nov  7 16:45 crypto
-r--r--r--   1 root  root               0 Nov  7 16:45 devices
-r--r--r--   1 root  root               0 Nov  7 07:07 diskstats
-r--r--r--   1 root  root               0 Nov  7 16:45 dma
dr-xr-xr-x   4 root  root               0 Nov  7 16:45 driver
-r--r--r--   1 root  root               0 Nov  7 16:45 execdomains
-r--r--r--   1 root  root               0 Nov  7 16:45 fb
-r--r--r--   1 root  root               0 Nov  7 07:06 filesystems
dr-xr-xr-x   5 root  root               0 Nov  7 07:07 fs
-r--r--r--   1 root  root               0 Nov  7 07:06 interrupts
-r--r--r--   1 root  root               0 Nov  7 07:07 iomem
-r--r--r--   1 root  root               0 Nov  7 16:45 ioports
dr-xr-xr-x  28 root  root               0 Nov  7 07:06 irq
-r--r--r--   1 root  root               0 Nov  7 07:07 kallsyms
-r--------   1 root  root 140737477890048 Nov  7 07:06 kcore
-r--r--r--   1 root  root               0 Nov  7 16:45 keys
-r--r--r--   1 root  root               0 Nov  7 16:45 key-users
-r--------   1 root  root               0 Nov  7 16:45 kmsg
-r--------   1 root  root               0 Nov  7 16:45 kpagecgroup
-r--------   1 root  root               0 Nov  7 16:45 kpagecount
-r--------   1 root  root               0 Nov  7 16:45 kpageflags
-rw-r--r--   1 root  root               0 Nov  7 16:45 latency_stats
-r--r--r--   1 root  root               0 Nov  7 07:07 loadavg
-r--r--r--   1 root  root               0 Nov  7 16:45 loadavg_bt
-r--r--r--   1 root  root               0 Nov  7 16:45 locks
-r--r--r--   1 root  root               0 Nov  7 07:17 mdstat
dr-xr-xr-x   2 root  root               0 Nov  7 16:45 megaraid
-r--r--r--   1 root  root               0 Nov  7 07:06 meminfo
-r--r--r--   1 root  root               0 Nov  7 16:45 misc
-r--r--r--   1 root  root               0 Nov  7 16:45 module_md5_list
-r--r--r--   1 root  root               0 Nov  7 07:07 modules
lrwxrwxrwx   1 root  root              11 Nov  7 07:06 mounts -> self/mounts
dr-xr-xr-x   4 root  root               0 Nov  7 16:45 mpt
-rw-r--r--   1 root  root               0 Nov  7 16:45 mtrr
lrwxrwxrwx   1 root  root               8 Nov  7 07:06 net -> self/net
-r--------   1 root  root               0 Nov  7 16:45 pagetypeinfo
-r--r--r--   1 root  root               0 Nov  7 07:06 partitions
dr-xr-xr-x   5 root  root               0 Nov  7 07:07 pressure
dr-xr-xr-x   3 root  root               0 Nov  7 16:45 rue
-r--r--r--   1 root  root               0 Nov  7 16:45 sched_debug
-r--r--r--   1 root  root               0 Nov  7 16:45 schedstat
dr-xr-xr-x   5 root  root               0 Nov  7 16:45 scsi
lrwxrwxrwx   1 root  root               0 Nov  7 07:06 self -> 26694
-r--------   1 root  root               0 Nov  7 16:45 slabinfo
dr-xr-xr-x   5 root  root               0 Nov  7 16:45 sli
-r--r--r--   1 root  root               0 Nov  7 16:45 softirqs
-r--r--r--   1 root  root               0 Nov  7 07:07 stat
-r--r--r--   1 root  root               0 Nov  7 07:06 swaps
dr-xr-xr-x   1 root  root               0 Nov  7 07:06 sys
--w-------   1 root  root               0 Nov  7 16:45 sysrq-trigger
dr-xr-xr-x   5 root  root               0 Nov  7 16:45 sysvipc
lrwxrwxrwx   1 root  root               0 Nov  7 07:06 thread-self -> 26694/task/26694
-r--------   1 root  root               0 Nov  7 16:45 timer_list
dr-xr-xr-x   5 root  root               0 Nov  7 16:45 tkernel
dr-xr-xr-x   6 root  root               0 Nov  7 07:07 tty
-r--r--r--   1 root  root               0 Nov  7 07:06 uptime
-r--r--r--   1 root  root               0 Nov  7 07:07 version
-r--------   1 root  root               0 Nov  7 16:45 vmallocinfo
-r--r--r--   1 root  root               0 Nov  7 07:07 vmstat
-r--r--r--   1 root  root               0 Nov  7 16:45 zoneinfo
```

比如，访问进程`13323`下的`limits`文件，能获取到本进程的资源限制

```BASH
[root@VM-X-X-tencentos bcc]#  cat /proc/13323/limits
Limit                     Soft Limit           Hard Limit           Units     
Max cpu time              unlimited            unlimited            seconds   
Max file size             unlimited            unlimited            bytes     
Max data size             unlimited            unlimited            bytes     
Max stack size            8388608              unlimited            bytes     
Max core file size        0                    0                    bytes     
Max resident set          unlimited            unlimited            bytes     
Max processes             62825                62825                processes 
Max open files            1024                 524288               files     
Max locked memory         8388608              8388608              bytes     
Max address space         unlimited            unlimited            bytes     
Max file locks            unlimited            unlimited            locks     
Max pending signals       62825                62825                signals   
Max msgqueue size         819200               819200               bytes     
Max nice priority         0                    0                    
Max realtime priority     0                    0                    
Max realtime timeout      unlimited            unlimited            us    
```

使用bcc的trace[工具](https://github.com/iovisor/bcc/blob/master/tools/trace.py)跟踪上面的`cat`过程，可以看到最终调用了`proc_pid_limits`函数

```BASH
PID     TID     COMM            FUNC
21450   21450   cat             proc_pid_limits
        proc_pid_limits+0x1 
        seq_read+0xe5 
        __vfs_read+0x1b 
        vfs_read+0x8e       #调用链vfs_read
        sys_read+0x55 
        do_syscall_64+0x73 
        entry_SYSCALL_64_after_hwframe+0x3d 
```

![proc_limits](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/vfs/procs/proc/proc_pid_limits-flow.png)


再列举一个例子，访问`ls /proc/13323/fd`可以知道当前进程`13323`打开了哪些fd：

```BASH
[root@VM-X-X-tencentos proc]# ll -ath /proc/13323/fd
total 0
lrwx------ 1 root root 64 Feb 10 10:43 1 -> /dev/pts/1
lrwx------ 1 root root 64 Feb 10 10:43 2 -> /dev/pts/1
lr-x------ 1 root root 64 Feb 10 10:43 3 -> 'pipe:[2640532256]'
lrwx------ 1 root root 64 Feb 10 10:43 4 -> 'socket:[2640532254]'
l-wx------ 1 root root 64 Feb 10 10:43 5 -> 'pipe:[2640532256]'
lr-x------ 1 root root 64 Feb 10 10:43 6 -> /etc/sudoers
lrwx------ 1 root root 64 Feb 10 10:43 7 -> 'socket:[2640532270]'
dr-x------ 2 root root  8 Feb 10 10:43 .
lrwx------ 1 root root 64 Feb 10 10:43 0 -> /dev/pts/1
dr-xr-xr-x 9 root root  0 Feb 10 10:43 ..
```

上面的命令对应用户态内核态切换如下图：

![fd](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/vfs/procs/proc/procfs-basic.png)

####    procs 详情
`/proc`下面主要包含如下内容：

1.  静态生成的（系统初始化生成）,比如`/proc/fs`、`/proc/fb`、`/proc/net`等，这部分子目录在系统初始化时候，应该挂载在proc目录对应的`proc_dir_entry`链表下
2.  `.`、`..`子目录，分别是对当前目录和父目录的链接
3.  `/proc/`下由数字组成的进程子目录是每次读取proc内容**动态生成**，即动态遍历当前进程列表形成；以及每进程子目录下子目录/子文件的形成

####	proc下的主要目录&&文件
TODO

##  0x02 procfs 的内核视角

####    pseudo FS

####    核心数据结构
-   [`proc_dir_entry`](https://elixir.bootlin.com/linux/v4.11.6/source/fs/proc/internal.h#L33)：对`/proc`目录的描述
-   [`pid_entry`](https://elixir.bootlin.com/linux/v4.11.6/source/fs/proc/base.c#L115)：proc目录下面的进程子目录，是对应与每一个进程ID pid目录的描述

####    proc_dir_entry
`proc_dir_entry`的结构如下：

```CPP
struct proc_dir_entry {
	unsigned int low_ino;
	umode_t mode;
	nlink_t nlink;
	kuid_t uid;
	kgid_t gid;
	loff_t size;
	const struct inode_operations *proc_iops;     // 文件inode操作函数
	const struct file_operations *proc_fops;     // 文件操作函数
	struct proc_dir_entry *parent;
	struct rb_root subdir;
	struct rb_node subdir_node;
	void *data;
	atomic_t count;		/* use count */
	atomic_t in_use;	/* number of callers into module in progress; */
			/* negative -> it's going away RSN */
	struct completion *pde_unload_completion;
	struct list_head pde_openers;	/* who did ->open, but not ->release */
	spinlock_t pde_unload_lock; /* proc_fops checks and pde_users bumps */
	u8 namelen;
	char name[];
};
```

注意，在高版本的内核中，`subdir`、`subdir_node`已经调整为红黑树的实现了，2.6的内核实现是链表。数据结构`proc_dir_entry`在内核中代表了一个proc入口，在procfs中表现为一个文件，可以在这个结构体中看到一些文件特有的属性成员，如`uid`、`gid`、`mode`、`name`等

`proc_dir_entry`的本质是什么？

TODO

####    pid_entry
```CPP
struct pid_entry {
	const char *name;
	unsigned int len;
	umode_t mode;
	const struct inode_operations *iop;
	const struct file_operations *fop;
	union proc_op op;
};
```

`pid_entry`下面所有的文件的操作方法都定义在[`tgid_base_stuff`](https://elixir.bootlin.com/linux/v4.11.6/source/fs/proc/base.c#L2843)结构中

```CPP
static const struct pid_entry tgid_base_stuff[] = {
	DIR("task",       S_IRUGO|S_IXUGO, proc_task_inode_operations, proc_task_operations),
	DIR("fd",         S_IRUSR|S_IXUSR, proc_fd_inode_operations, proc_fd_operations),
	DIR("map_files",  S_IRUSR|S_IXUSR, proc_map_files_inode_operations, proc_map_files_operations),
	DIR("fdinfo",     S_IRUSR|S_IXUSR, proc_fdinfo_inode_operations, proc_fdinfo_operations),
	DIR("ns",	  S_IRUSR|S_IXUGO, proc_ns_dir_inode_operations, proc_ns_dir_operations),
#ifdef CONFIG_NET
	DIR("net",        S_IRUGO|S_IXUGO, proc_net_inode_operations, proc_net_operations),
#endif
	REG("environ",    S_IRUSR, proc_environ_operations),
	REG("auxv",       S_IRUSR, proc_auxv_operations),
	ONE("status",     S_IRUGO, proc_pid_status),
	ONE("personality", S_IRUSR, proc_pid_personality),
	ONE("limits",	  S_IRUGO, proc_pid_limits),
#ifdef CONFIG_SCHED_DEBUG
	REG("sched",      S_IRUGO|S_IWUSR, proc_pid_sched_operations),
#endif
#ifdef CONFIG_SCHED_AUTOGROUP
	REG("autogroup",  S_IRUGO|S_IWUSR, proc_pid_sched_autogroup_operations),
#endif
	REG("comm",      S_IRUGO|S_IWUSR, proc_pid_set_comm_operations),
#ifdef CONFIG_HAVE_ARCH_TRACEHOOK
	ONE("syscall",    S_IRUSR, proc_pid_syscall),
#endif
	REG("cmdline",    S_IRUGO, proc_pid_cmdline_ops),
	ONE("stat",       S_IRUGO, proc_tgid_stat),
	ONE("statm",      S_IRUGO, proc_pid_statm),
	REG("maps",       S_IRUGO, proc_pid_maps_operations),
#ifdef CONFIG_NUMA
	REG("numa_maps",  S_IRUGO, proc_pid_numa_maps_operations),
#endif
	REG("mem",        S_IRUSR|S_IWUSR, proc_mem_operations),
	LNK("cwd",        proc_cwd_link),
	LNK("root",       proc_root_link),
	LNK("exe",        proc_exe_link),
	REG("mounts",     S_IRUGO, proc_mounts_operations),
	REG("mountinfo",  S_IRUGO, proc_mountinfo_operations),
	REG("mountstats", S_IRUSR, proc_mountstats_operations),
#ifdef CONFIG_PROC_PAGE_MONITOR
	REG("clear_refs", S_IWUSR, proc_clear_refs_operations),
	REG("smaps",      S_IRUGO, proc_pid_smaps_operations),
	REG("pagemap",    S_IRUSR, proc_pagemap_operations),
#endif
#ifdef CONFIG_SECURITY
	DIR("attr",       S_IRUGO|S_IXUGO, proc_attr_dir_inode_operations, proc_attr_dir_operations),
#endif
#ifdef CONFIG_KALLSYMS
	ONE("wchan",      S_IRUGO, proc_pid_wchan),
#endif
#ifdef CONFIG_STACKTRACE
	ONE("stack",      S_IRUSR, proc_pid_stack),
#endif
#ifdef CONFIG_SCHED_INFO
	ONE("schedstat",  S_IRUGO, proc_pid_schedstat),
#endif
#ifdef CONFIG_LATENCYTOP
	REG("latency",  S_IRUGO, proc_lstats_operations),
#endif
#ifdef CONFIG_PROC_PID_CPUSET
	ONE("cpuset",     S_IRUGO, proc_cpuset_show),
#endif
#ifdef CONFIG_CGROUPS
	ONE("cgroup",  S_IRUGO, proc_cgroup_show),
#endif
	ONE("oom_score",  S_IRUGO, proc_oom_score),
	REG("oom_adj",    S_IRUGO|S_IWUSR, proc_oom_adj_operations),
	REG("oom_score_adj", S_IRUGO|S_IWUSR, proc_oom_score_adj_operations),
#ifdef CONFIG_AUDITSYSCALL
	REG("loginuid",   S_IWUSR|S_IRUGO, proc_loginuid_operations),
	REG("sessionid",  S_IRUGO, proc_sessionid_operations),
#endif
#ifdef CONFIG_FAULT_INJECTION
	REG("make-it-fail", S_IRUGO|S_IWUSR, proc_fault_inject_operations),
#endif
#ifdef CONFIG_ELF_CORE
	REG("coredump_filter", S_IRUGO|S_IWUSR, proc_coredump_filter_operations),
#endif
#ifdef CONFIG_TASK_IO_ACCOUNTING
	ONE("io",	S_IRUSR, proc_tgid_io_accounting),
#endif
#ifdef CONFIG_HARDWALL
	ONE("hardwall",   S_IRUGO, proc_pid_hardwall),
#endif
#ifdef CONFIG_USER_NS
	REG("uid_map",    S_IRUGO|S_IWUSR, proc_uid_map_operations),
	REG("gid_map",    S_IRUGO|S_IWUSR, proc_gid_map_operations),
	REG("projid_map", S_IRUGO|S_IWUSR, proc_projid_map_operations),
	REG("setgroups",  S_IRUGO|S_IWUSR, proc_setgroups_operations),
#endif
#if defined(CONFIG_CHECKPOINT_RESTORE) && defined(CONFIG_POSIX_TIMERS)
	REG("timers",	  S_IRUGO, proc_timers_operations),
#endif
	REG("timerslack_ns", S_IRUGO|S_IWUGO, proc_pid_set_timerslack_ns_operations),
};
```

`tgid_base_stuff`结构中的几个宏定义：

1、`DIR`：目录条目，`DIR("task", S_IRUGO|S_IXUGO, proc_task_inode_operations, proc_task_operations)`表示创建目录`/proc/<pid>/task/`，该目录包含该进程的所有线程信息，参数分别表示目录名、权限、inode操作、文件操作

```CPP
static const struct file_operations proc_task_operations = {
	.read		= generic_read_dir,
	.iterate_shared	= proc_task_readdir,	//proc_task_readdir：对应的目录遍历方法实现
	.llseek		= generic_file_llseek,
};

static const struct inode_operations proc_task_inode_operations = {
	.lookup		= proc_task_lookup,
	.getattr	= proc_task_getattr,
	.setattr	= proc_setattr,
	.permission	= proc_pid_permission,
};
```

又如`DIR("fd",S_IRUSR|S_IXUSR, proc_fd_inode_operations, proc_fd_operations)`，该目录表示某个进程打开的所有fd列表

```BASH
[root@VM-X-X-tencentos ~]# ls /proc/1081/fd
0  1  2  3  4
```

```CPP
const struct inode_operations proc_fd_inode_operations = {
	.lookup		= proc_lookupfd,	//lookup 对应的lookup方法实现
	.permission	= proc_fd_permission,
	.setattr	= proc_setattr,
};

const struct file_operations proc_fd_operations = {
	.read		= generic_read_dir,
	.iterate_shared	= proc_readfd,
	.llseek		= generic_file_llseek,
};
```

2、`REG`：表示常规文件条目，`REG("environ", S_IRUSR, proc_environ_operations)`，创建一个常规文件，如`/proc/<pid>/environ` 文件，显示进程的环境变量，参数分别为文件名、权限、文件操作结构体

```CPP
//https://elixir.bootlin.com/linux/v4.11.6/source/fs/proc/base.c#L974
static const struct file_operations proc_environ_operations = {
	.open		= environ_open,
	.read		= environ_read,
	.llseek		= generic_file_llseek,
	.release	= mem_release,
};
```

3、`ONE`：单一文件条目，`ONE("status", S_IRUGO, proc_pid_status)`，创建一个只读的常规文件，使用简化的回调函数，如`/proc/<pid>/status`文件是显示进程状态信息，参数分别是文件名、权限、数据生成函数指针。与 `REG` 的区别是`ONE` 使用简单的 `C`，而 `REG` 需要完整的文件操作结构体

```CPP
int proc_pid_status(struct seq_file *m, struct pid_namespace *ns,
			struct pid *pid, struct task_struct *task)
{
	struct mm_struct *mm = get_task_mm(task);

	task_name(m, task);
	task_state(m, ns, pid, task);

	if (mm) {
		task_mem(m, mm);
		mmput(mm);
	}
	task_sig(m, task);
	task_cap(m, task);
	task_seccomp(m, task);
	task_cpus_allowed(m, task);
	cpuset_task_status_allowed(m, task);
	task_context_switch_counts(m, task);
	return 0;
}
```

4、`LNK`：符号链接条目，`LNK("cwd", proc_cwd_link)`表示创建一个符号链接，如`/proc/<pid>/cwd` 链接指向进程的当前工作目录，参数分别为链接名、链接目标生成函数

```BASH
[root@VM-X-X-tencentos ~]# ll -alrth  /proc/1081/
......
lrwxrwxrwx   1 root root 0 Nov  7 07:07 root -> /
lrwxrwxrwx   1 root root 0 Nov  7 07:07 cwd -> /

[root@VM-X-X-tencentos ~]# ls /proc/1081/cwd/
bin  boot  data  dev  etc  home  lib  lib64  lost+found  media  mnt  opt  proc  root  run  sbin  srv  sys  tmp  usr  var
```

```CPP
static int proc_cwd_link(struct dentry *dentry, struct path *path)
{
	struct task_struct *task = get_proc_task(d_inode(dentry));
	int result = -ENOENT;

	if (task) {
		task_lock(task);
		if (task->fs) {
			get_fs_pwd(task->fs, path); //返回task->fs指向的目录，保存在path中
			result = 0;
		}
		task_unlock(task);
		put_task_struct(task);
	}
	return result;
}

static inline void get_fs_pwd(struct fs_struct *fs, struct path *pwd)
{
	spin_lock(&fs->lock);
	*pwd = fs->pwd;
	path_get(pwd);
	spin_unlock(&fs->lock);
}
```

上面介绍的回调函数，如`struct file_operations/inode_operations`等，即当触发内核代码调用VFS架构下的`struct file/inode`的操作时，会调用响应的操作函数（如`file_operations`的`open/read/llseek`等），未定义的操作即不支持

####    小结
当程序读取`/proc`下面的文件时，内核的处理如下：

![proc-flow](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/vfs/procs/proc/read_proc_apis.png)

##  0x03 proc_dir_entry 主要功能分析
本小节主要基于内核代码，分析下`/proc`及子目录的初始化流程

1、内核初始化创建`/proc`目录，[代码](https://elixir.bootlin.com/linux/v4.11.6/source/init/main.c#L488)

```CPP
asmlinkage void __init start_kernel(void)
{
	//...
    proc_root_init();
	//...
}
```

2、`proc_root_init->register_filesystem->proc_sys_init`

```CPP
void __init proc_root_init(void)
{
	int err;

	proc_init_inodecache();
	set_proc_pid_nlink();
	err = register_filesystem(&proc_fs_type);	//  注册proc文件系统
	if (err)
		return;

	proc_self_init();
	proc_thread_self_init();
	proc_symlink("mounts", NULL, "self/mounts");	 // 创建 mounts 符号链接文件

	proc_net_init();							 // 创建 net符号链接及内部目录树结构

#ifdef CONFIG_SYSVIPC
	proc_mkdir("sysvipc", NULL);
#endif
	proc_mkdir("fs", NULL);						// 创建 fs 目录
	proc_mkdir("driver", NULL);					// 创建 drivers 目录
	proc_create_mount_point("fs/nfsd"); /* somewhere for the nfsd filesystem to be mounted */
#if defined(CONFIG_SUN_OPENPROMFS) || defined(CONFIG_SUN_OPENPROMFS_MODULE)
	/* just give it a mountpoint */
	proc_create_mount_point("openprom");
#endif
	proc_tty_init();
	proc_mkdir("bus", NULL);
	proc_sys_init();							// 创建sys目录并初始化
}

//注册proc文件系统
//https://elixir.bootlin.com/linux/v4.11.6/source/fs/proc/root.c#L116
static struct file_system_type proc_fs_type = {
	.name		= "proc",
	.mount		= proc_mount,	//挂载入口
	.kill_sb	= proc_kill_sb,
	.fs_flags	= FS_USERNS_MOUNT,
};
```

3、挂载procfs入口：`proc_mount->proc_fill_super`，将procfs文件系统挂载到内核全局的VFS树中，调用`proc_fill_super`完成超级快的初始化工作

```CPP
//https://elixir.bootlin.com/linux/v4.11.6/source/fs/proc/root.c#L88
static struct dentry *proc_mount(struct file_system_type *fs_type,
	int flags, const char *dev_name, void *data)
{
	struct pid_namespace *ns;

	if (flags & MS_KERNMOUNT) {
		ns = data;
		data = NULL;
	} else {
		ns = task_active_pid_ns(current);
	}

	return mount_ns(fs_type, flags, data, ns, ns->user_ns, proc_fill_super);
}
```

```CPP
int proc_fill_super(struct super_block *s, void *data, int silent)
{
	struct pid_namespace *ns = get_pid_ns(s->s_fs_info);
	struct inode *root_inode;
	int ret;

	if (!proc_parse_options(data, ns))
		return -EINVAL;

	/* User space would break if executables or devices appear on proc */
	s->s_iflags |= SB_I_USERNS_VISIBLE | SB_I_NOEXEC | SB_I_NODEV;
	s->s_flags |= MS_NODIRATIME | MS_NOSUID | MS_NOEXEC;
	s->s_blocksize = 1024;
	s->s_blocksize_bits = 10;	// 必须是10,2^10=1024
	s->s_magic = PROC_SUPER_MAGIC;	 //magic number，宏的具体数字为0x9fa0
	s->s_op = &proc_sops;		// 具体的超级块操作,主要涉及的是索引块的操作
	s->s_time_gran = 1;

	s->s_stack_depth = FILESYSTEM_MAX_STACK_DEPTH;
	
	pde_get(&proc_root);
	root_inode = proc_get_inode(s, &proc_root);	// 转换为vfs具体能识别的索引节点
	if (!root_inode) {
		pr_err("proc_fill_super: get root inode failed\n");
		return -ENOMEM;
	}
	//https://elixir.bootlin.com/linux/v4.11.6/source/fs/dcache.c#L1855
	//初始化根节点，并且与super_block进行关联！
	s->s_root = d_make_root(root_inode);
	if (!s->s_root) {
		pr_err("proc_fill_super: allocate dentry failed\n");
		return -ENOMEM;
	}

	ret = proc_setup_self(s);
	if (ret) {
		return ret;
	}
	return proc_setup_thread_self(s);
}

enum {
    PROC_ROOT_INO = 1,
};

//https://elixir.bootlin.com/linux/v4.11.6/source/fs/proc/root.c#L204
struct proc_dir_entry proc_root = {
	.low_ino	= PROC_ROOT_INO, 	  // 根的索引节点号
	.namelen	= 5, 					 // 根文件名长度、文件名
	.mode		= S_IFDIR | S_IRUGO | S_IXUGO, 
	.nlink		= 2, 
	.count		= ATOMIC_INIT(1),
	.proc_iops	= &proc_root_inode_operations, 	// 根文件的具体索引节点操作
	.proc_fops	= &proc_root_operations,		// 根文件支持的文件操作
	.parent		= &proc_root,
	.subdir		= RB_ROOT,
	.name		= "/proc",
};
```

上面的`proc_root`节点不仅包含正常的文件及目录，还要管理进程指定的pid文件，`proc_root`必须能够处理inode和file，`proc_root_inode_operations`与`proc_root_operations`的定义如下：

```CPP
static struct file_operations proc_root_operations = {
    .read        = generic_read_dir,               
    .readdir     = proc_root_readdir,		//目录遍历
};
 
/*
 * proc root can do almost nothing..
 */
static struct inode_operations proc_root_inode_operations = {
    .lookup     = proc_root_lookup,			//对应vfs架构中的real_lookup函数
    .getattr    = proc_root_getattr,
};
```

####	proc_root_lookup：proc下的inode查找
当用户空间访问proc文件的时候，vfs就会调用`real_lookup()`，它就会调用`inode_operations`中的`proc_root_lookup`函数，实际上就是调用`proc_root_lookup()`函数

```CPP
//https://elixir.bootlin.com/linux/v4.11.6/source/fs/proc/root.c#L204
static struct dentry *proc_root_lookup(struct inode * dir, struct dentry * dentry, unsigned int flags)
{
	 // 先查找进程id相关的文件
	if (!proc_pid_lookup(dir, dentry, flags)){
		return NULL;
	}
	// 再查找内核运行状态的文件
	return proc_lookup(dir, dentry, flags);
}

//https://elixir.bootlin.com/linux/v4.11.6/source/fs/proc/generic.c#L251
struct dentry *proc_lookup(struct inode *dir, struct dentry *dentry,
		unsigned int flags)
{
	return proc_lookup_de(PDE(dir), dir, dentry);
}
```

`proc_pid_lookup`与`proc_lookup`的实现：

```CPP
//在指定的pid文件夹中查找dentry是否存在
struct dentry *proc_pid_lookup(struct inode *dir, struct dentry * dentry, unsigned int flags)
{
	int result = -ENOENT;
	struct task_struct *task;
	unsigned tgid;
	struct pid_namespace *ns;

	// 检查目录是否为数字（快速失败）
	tgid = name_to_int(&dentry->d_name);
	if (tgid == ~0U)
		goto out;

	ns = dentry->d_sb->s_fs_info;
	rcu_read_lock();
	//通过pid查找到指定的task
	task = find_task_by_pid_ns(tgid, ns);
	if (task)
		get_task_struct(task);
	rcu_read_unlock();
	if (!task)
		goto out;
	
	//生成一个新索引节点，并进行缓存
	result = proc_pid_instantiate(dir, dentry, task, NULL);
	put_task_struct(task);
out:
	return ERR_PTR(result);
}

//proc_lookup->proc_lookup_de
//https://elixir.bootlin.com/linux/v4.11.6/source/fs/proc/generic.c#L230
struct dentry *proc_lookup_de(struct proc_dir_entry *de, struct inode *dir,
		struct dentry *dentry)
{
	struct inode *inode;

	read_lock(&proc_subdir_lock);
	// 从dentry中提取出具体的proc_dir_entry
	de = pde_subdir_find(de, dentry->d_name.name, dentry->d_name.len);
	if (de) {
		//找到了
		pde_get(de);
		read_unlock(&proc_subdir_lock);
		inode = proc_get_inode(dir->i_sb, de);	// 获取对应的inode
		if (!inode)
			return ERR_PTR(-ENOMEM);
		d_set_d_op(dentry, &simple_dentry_operations); 	//老套路了，加入到dentry cache中
		d_add(dentry, inode);
		return NULL;
	}
	read_unlock(&proc_subdir_lock);
	return ERR_PTR(-ENOENT);
}
```

为什么`proc_root_lookup`中的设计要优先进程id其次再内核呢？这是考虑到进程目录数量远大于内核文件数量，先匹配高频访问如进程相关的查找（`ps/top` 等）比内核状态文件访问更频繁，其次快速失败机制如果查找的不是数字（非PID），`proc_pid_lookup` 会快速返回失败以切换到查找内核文件

####	file_operations：proc_root_readdir
`proc_root_operations`对应的`.readdir`实现为`proc_root_readdir`，该函数是procfs 文件系统根目录`/proc/`的目录遍历实现，它的作用是控制如何列出 `/proc` 目录下的内容

```CPP
#define FIRST_PROCESS_ENTRY 256

//https://elixir.bootlin.com/linux/v4.11.6/source/fs/proc/root.c#L170
static int proc_root_readdir(struct file *file, struct dir_context *ctx)
{
    // 第一阶段：读取内核状态文件（静态文件）
    if (ctx->pos < FIRST_PROCESS_ENTRY) {
        int error = proc_readdir(file, ctx);  // 读取内核相关文件
        if (unlikely(error <= 0))
            return error;  // 出错或读取完成
        ctx->pos = FIRST_PROCESS_ENTRY;  // 切换到进程条目阶段
    }
    
    // 第二阶段：读取进程目录
    return proc_pid_readdir(file, ctx);  // 读取进程PID目录
}
```

系统中对于一个目录有多种读取子目录的方式（如`ls`和`ls -al`显示的结果不同），这是由传入过程中对`file->f_ops`设定不同的偏移决定的。对于`/proc`根目录而言：

-	`f_ops = 0`：为`.`目录链接，链接到自身
-	`f_ops = 1`：为`..`目录链接，链接到父目录
-	`f_ops` 如果是在`2~FIRST_PROCESS_ENTRY-1`之间：表示`/proc/`下的静态目录或者静态文件
-	`f_ops` 如果是在`FIRST_PROCESS_ENTRY~FIRST_PROCESS_ENTRY+ ARRAY_SIZE(proc_base_stuff)-1`：为`self`子目录内容
-	`f_ops = FIRST_PROCESS_ENTRY+ ARRAY_SIZE(proc_base_stuff)`：为`init_task`即`0`号初始进程
-	`f_pos = PID_MAX_LIMIT + TGID_OFFSET`：标识目录遍历结束

继续分析`proc_pid_readdir`的实现，该函数用于列出 `/proc`目录时生成进程列表，包括

-	特殊符号链接：`self`、`thread-self`
-	所有进程目录：`/proc/1`、 `/proc/2`等

```CPP
int proc_pid_readdir(struct file *file, struct dir_context *ctx)
{
	struct tgid_iter iter;
	// pid_namespace 为pid结构的命名空间
	struct pid_namespace *ns = file_inode(file)->i_sb->s_fs_info;
	loff_t pos = ctx->pos;

	if (pos >= PID_MAX_LIMIT + TGID_OFFSET)
		return 0;

	if (pos == TGID_OFFSET - 2) {
		struct inode *inode = d_inode(ns->proc_self);
		//self
		if (!dir_emit(ctx, "self", 4, inode->i_ino, DT_LNK))
			return 0;
		ctx->pos = pos = pos + 1;
	}
	if (pos == TGID_OFFSET - 1) {
		struct inode *inode = d_inode(ns->proc_thread_self);
		if (!dir_emit(ctx, "thread-self", 11, inode->i_ino, DT_LNK))
			return 0;
		ctx->pos = pos = pos + 1;
	}
	iter.tgid = pos - TGID_OFFSET;
	iter.task = NULL;
	// next_tgid(ns, iter) 来寻找每一个pid（对应ns命名空间内的）
	for (iter = next_tgid(ns, iter);
	     iter.task;
	     iter.tgid += 1, iter = next_tgid(ns, iter)) {
		char name[PROC_NUMBUF];
		int len;

		cond_resched();
		if (!has_pid_permissions(ns, iter.task, HIDEPID_INVISIBLE))
			continue;

		len = snprintf(name, sizeof(name), "%d", iter.tgid);
		ctx->pos = iter.tgid + TGID_OFFSET;
		if (!proc_fill_cache(file, ctx, name, len,
				     proc_pid_instantiate, iter.task, NULL)) {
			put_task_struct(iter.task);
			return 0;
		}
	}
	// 标志着目录遍历结束
	ctx->pos = PID_MAX_LIMIT + TGID_OFFSET;
	return 0;
}
```


```CPP
static struct tgid_iter next_tgid(struct pid_namespace *ns, struct tgid_iter iter)
{
	struct pid *pid;

	if (iter.task)
		put_task_struct(iter.task);
	rcu_read_lock();
retry:
	iter.task = NULL;
	pid = find_ge_pid(iter.tgid, ns);
	if (pid) {
		iter.tgid = pid_nr_ns(pid, ns);
		iter.task = pid_task(pid, PIDTYPE_PID);

		if (!iter.task || !has_group_leader_pid(iter.task)) {
			iter.tgid += 1;
			goto retry;
		}
		get_task_struct(iter.task);
	}
	rcu_read_unlock();
	return iter;
}
```

##  0x04 pid_entry主要功能分析

##  0x05 proc 函数钩子实例分析（进程属性相关）

####    proc_pid_limit的实现
`/proc/x/limits`实时反映当前进程的资源限制

```BASH
[root@VM-X-X-centos ]# cat /proc/4124608/limits 
```

![proc_pid_limits](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/vfs/procs/proc/proc_pid_limits.png)


[`proc_pid_limits`](https://elixir.bootlin.com/linux/v4.11.6/source/fs/proc/base.c#L573)

```cpp
/* Display limits for a process */
static int proc_pid_limits(struct seq_file *m, struct pid_namespace *ns,
			   struct pid *pid, struct task_struct *task)
{
	unsigned int i;
	unsigned long flags;

	struct rlimit rlim[RLIM_NLIMITS];

	if (!lock_task_sighand(task, &flags))
		return 0;
	memcpy(rlim, task->signal->rlim, sizeof(struct rlimit) * RLIM_NLIMITS);
	unlock_task_sighand(task, &flags);

	/*
	 * print the file header
	 */
       seq_printf(m, "%-25s %-20s %-20s %-10s\n",
		  "Limit", "Soft Limit", "Hard Limit", "Units");

	for (i = 0; i < RLIM_NLIMITS; i++) {
		if (rlim[i].rlim_cur == RLIM_INFINITY)
			seq_printf(m, "%-25s %-20s ",
				   lnames[i].name, "unlimited");
		else
			seq_printf(m, "%-25s %-20lu ",
				   lnames[i].name, rlim[i].rlim_cur);

		if (rlim[i].rlim_max == RLIM_INFINITY)
			seq_printf(m, "%-20s ", "unlimited");
		else
			seq_printf(m, "%-20lu ", rlim[i].rlim_max);

		if (lnames[i].unit)
			seq_printf(m, "%-10s\n", lnames[i].unit);
		else
			seq_putc(m, '\n');
	}

	return 0;
}
```

####    proc_pid_cmdline的实现

`/proc/x/cmdline`记录了进程启动时的完整命令行参数

```BASH
[root@VM-X-X-centos X]# cat /proc/4124608/cmdline 
sudosu-root
```

对应的VFS read实现方法`proc_pid_cmdline`：通过该方法获取进程对应的cmdline文件的信息

```CPP
static int proc_pid_cmdline(struct task_struct *task, char * buffer)
{
    int res = 0;
    unsigned int len;
    struct mm_struct *mm = get_task_mm(task);
    if (!mm)
        goto out;
    if (!mm->arg_end)
        goto out_mm;    /* Shh! No looking before we're done */
 
    // 获取所有参数的个数
    len = mm->arg_end - mm->arg_start;
  
    if (len > PAGE_SIZE)
        len = PAGE_SIZE;
  
    // 获取第一个参数
    res = access_process_vm(task, mm->arg_start, buffer, len, 0);
 
    // If the nul at the end of args has been overwritten, then
    // assume application is using setproctitle(3).
    // 循环获取所有的参数
    if (res > 0 && buffer[res-1] != '\0' && len < PAGE_SIZE) {
        len = strnlen(buffer, res);
        if (len < res) {
            res = len;
        } else {
            len = mm->env_end - mm->env_start;
            if (len > PAGE_SIZE - res)
                len = PAGE_SIZE - res;
            // 获取参数,拼接参数得到cmdline
            res += access_process_vm(task, mm->env_start, buffer+res, len, 0);
            res = strnlen(buffer, res);
        }
    }
out_mm:
    mmput(mm);
out:
    return res;
}
```

##	0x06	proc 函数钩子实例分析（进程内存相关）
先梳理下进程的内存统计的背景知识，业务进程使用的内存主要有以下几种情况（其中前两者算作进程的`RSS`，后两者属于page cache）

-	用户空间的匿名映射页（Anonymous pages in User Mode address spaces）：比如调用`malloc`分配的内存，以及使用`MAP_ANONYMOUS`的`mmap`等场景；当系统内存不够时，内核可以将这部分内存交换出去
-	用户空间的文件映射页（Mapped pages in User Mode address spaces）：包含map file和map tmpfs；前者比如指定文件的`mmap`，后者比如IPC共享内存；当系统内存不够时，内核可以回收这些页，但回收之前可能需要与文件同步数据
-	文件缓存（page in page cache of disk file）：发生在程序通过普通的`read`/`write`读写文件时，当系统内存不够时，内核可以回收这些页，但回收之前可能需要与文件同步数据
-	buffer pages，属于page cache：比如读取块设备文件

本小节汇总下与内存相关的几个proc文件

-	`/proc/[pid]/stat`
-	`/proc/[pid]/statm`

####	VSS/RSS/PSS/USS
-	`VSS`：（Virtual Set Size）虚拟耗用内存（包含共享库占用的内存），即是进程向系统申请的虚拟内存（包含共享库内存总数），即单个进程全部可访问的地址空间，其大小可能包括还尚未在内存中驻留的部分
-	`RSS`：（Resident Set Size）常驻内存大小，表示进程实际使用物理内存；是进程在 RAM 中实际保存的总内存（包含共享库占用的共享内存总数）。这个值经常用于进程内存监控，不过有一点需要关注：`RSS`包含了共享库占用的共享内存总数，然而实际上一个共享库仅会被加载到内存中一次，无论被多少个进程使用	
-	`PSS`：（Proportional Set Size） 实际使用的物理内存（比例分配共享库占用的内存），是单个进程运行时实际占用的物理内存（包含比例分配共享库占用的内存）。对比 `RSS` 来说，`PSS` 中的共享库内存是按照比例计算的（若一个共享库有 `N` 个进程使用，那么该库比例分配给 `PSS` 的大小为`1/N`），即`PSS` 明确的表示了单个进程在系统总内存中的实际使用量
-	`USS`：（Unique Set Size） 进程独自占用的物理内存（不包含共享库占用的内存），是进程实际独自占用的物理内存（不包含共享库占用的内存）。`USS` 揭示了单个进程运行中真实的内存增量大小。如果单个进程终止，`USS` 就是实际返还给系统的内存大小

一般来说内存占用大小有如下规律：`VSS >= RSS >= PSS >= USS`

####	do_task_stat的实现：/proc/[pid]/stat
`/proc/[pid]/stat`相关的输出字段：

```TEXT
(23) vsize  %lu
        Virtual memory size in bytes.
(24) rss  %ld
        Resident Set Size: number of pages the process has
        in real memory.  This is just the pages which count
        toward text, data, or stack space.  This does not
        include pages which have not been demand-loaded in,
        or which are swapped out.
```

对应的内核函数为[`do_task_stat`](https://elixir.bootlin.com/linux/v4.11.6/source/fs/proc/array.c#L393)，rss列的计算函数为[`get_mm_rss`](https://elixir.bootlin.com/linux/v4.11.6/source/fs/proc/array.c#L514)，可见`RSS`的计算包含了`MM_FILEPAGES`/`MM_ANONPAGES`/`MM_SHMEMPAGES`（单位：页）

```CPP
static inline unsigned long get_mm_rss(struct mm_struct *mm)
{
	return get_mm_counter(mm, MM_FILEPAGES) +
		get_mm_counter(mm, MM_ANONPAGES) +
		get_mm_counter(mm, MM_SHMEMPAGES);
}
```

注意到，`/proc/[pid]/stat`的`RSS`值与`/proc/[pid]/statm`输出的`RSS`是相等的

```BASH
[root@VM-x-x-centos memory]# cat /proc/3447782/stat | awk '{print "RSS(page):", $24}'
RSS(page): 22811
[root@VM-218-158-centos memory]# cat /proc/3447782/statm 
330060 22811 3727 1320 0 33217 0
```

####	 proc_pid_statm的实现
`/proc/${PID}/statm` 用于记录指定进程的内存使用情况，所有数值以 内存页（Page）为单位（`1` 页通常为 `4` KB），关联内核函数为[`proc_pid_statm`](https://elixir.bootlin.com/linux/v4.11.6/source/fs/proc/array.c#L586)，即当用户态读取 `/proc/${PID}/statm` 时，内核通过虚拟文件系统（VFS）触发该函数

```TEXT
Provides information about memory usage, measured in pages.
The columns are:

  size       (1) total program size
             (same as VmSize in /proc/[pid]/status)
  resident   (2) resident set size
             (same as VmRSS in /proc/[pid]/status)
  share      (3) shared pages (i.e., backed by a file)
  text       (4) text (code)
  lib        (5) library (unused in Linux 2.6)
  data       (6) data + stack
  dt         (7) dirty pages (unused in Linux 2.6)
```

```BASH
[root@VM-X-X-centos ~]# cat /proc/3447782/statm 
329932/size/ 21419/resident/ 2538/share/ 1320/text/ 0 32065/data/ 0

#size：进程的虚拟内存总量（包括物理内存、交换区、未映射内存等）
#resident：实际驻留物理内存的大小（RSS），即进程当前使用的物理内存
#share：与其他进程共享的物理内存（如共享库、共享内存段）
#text：代码段（可执行指令）占用的物理内存（所有进程共享同一程序的代码段时，此值可能重复计算）
#data：数据段（全局变量、静态数据）和堆栈占用的物理内存（反映进程的堆内存和栈内存使用）
```

`proc_pid_statm`函数的实现如下，注意[`get_mm_counter`](https://elixir.bootlin.com/linux/v4.11.6/source/include/linux/mm.h#L1448)函数的作用是读取`mm_struct`结构体的成员数组计数`mm->rss_stat.count[index]`

```CPP
static inline unsigned long get_mm_counter(struct mm_struct *mm, int member)
{
	long val = atomic_long_read(&mm->rss_stat.count[member]);

#ifdef SPLIT_RSS_COUNTING
	/*
	 * counter is updated in asynchronous manner and may go to minus.
	 * But it's never be expected number for users.
	 */
	if (val < 0)
		val = 0;
#endif
	return (unsigned long)val;
}

unsigned long task_statm(struct mm_struct *mm,
			 unsigned long *shared, unsigned long *text,
			 unsigned long *data, unsigned long *resident)
{
	*shared = get_mm_counter(mm, MM_FILEPAGES) +
			get_mm_counter(mm, MM_SHMEMPAGES);
	*text = (PAGE_ALIGN(mm->end_code) - (mm->start_code & PAGE_MASK))
								>> PAGE_SHIFT;
	*data = mm->data_vm + mm->stack_vm;
	*resident = *shared + get_mm_counter(mm, MM_ANONPAGES);
	return mm->total_vm;
}

int proc_pid_statm(struct seq_file *m, struct pid_namespace *ns,
			struct pid *pid, struct task_struct *task)
{
	unsigned long size = 0, resident = 0, shared = 0, text = 0, data = 0;
	struct mm_struct *mm = get_task_mm(task);

	if (mm) {
		size = task_statm(mm, &shared, &text, &data, &resident);
		mmput(mm);
	}
	/*
	 * For quick read, open code by putting numbers directly
	 * expected format is
	 * seq_printf(m, "%lu %lu %lu %lu 0 %lu 0\n",
	 *               size, resident, shared, text, data);
	 */
	seq_put_decimal_ull(m, "", size);
	seq_put_decimal_ull(m, " ", resident);
	seq_put_decimal_ull(m, " ", shared);
	seq_put_decimal_ull(m, " ", text);
	seq_put_decimal_ull(m, " ", 0);	//废弃
	seq_put_decimal_ull(m, " ", data);
	seq_put_decimal_ull(m, " ", 0);	//废弃
	seq_putc(m, '\n');

	return 0;
}
```

##  0x07  参考
-   [Linux进程网络流量统计方法及实现](https://zhuanlan.zhihu.com/p/49981590)
-   [使用 golang gopacket 实现进程级流量监控](https://github.com/rfyiamcool/notes/blob/main/netflow.md)
-   [从内核代码角度详解proc目录](https://blog.spoock.com/2019/10/26/proc-from-kernel/)
-   [Linux下/proc目录简介](https://blog.spoock.com/2019/10/08/proc/)
-   [Linux Procfs (一) /proc/* 文件实例解析](https://juejin.cn/post/7055321925463048228)
-	[A journey into the Linux proc filesystem](https://fernandovillalba.substack.com/p/a-journey-into-the-linux-proc-filesystem)
-	[聊聊 Linux 的内存统计](https://www.0xffffff.org/2019/07/17/42-linux-memory-monitor/)
-	[Linux中进程内存与cgroup内存的统计](https://hustcat.github.io/memory-usage-in-process-and-cgroup/?spm=a2c6h.12873639.article-detail.4.4db57092lEvNeV)
-	[proc_pid_statm(5) — Linux manual page](https://man7.org/linux/man-pages/man5/proc_pid_statm.5.html)