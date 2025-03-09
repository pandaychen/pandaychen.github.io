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

本文代码基于 [v4.11.6](https://elixir.bootlin.com/linux/v4.11.6/source/include) 版本

##  0x01    引言：访问/proc文件

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

##  0x02 procfs 的内核视角

####    pseudo FS

####    核心数据结构
-   [`proc_dir_entry`](https://elixir.bootlin.com/linux/v4.11.6/source/fs/proc/internal.h#L33)：对`/proc`目录的描述
-   [`pid_entry`](https://elixir.bootlin.com/linux/v4.11.6/source/fs/proc/base.c#L115)：proc目录下面的进程子目录，是对应与每一个进程ID pid目录的描述

####    proc_dir_entry
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

####    小结
当程序读取`/proc`下面的文件时，内核的处理如下：

![proc-flow](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/vfs/procs/proc/read_proc_apis.png)

##  0x03 proc_dir_entry 主要功能分析
本小节主要基于内核代码，分析下`/proc`及子目录的初始化流程

1、内核初始化创建`/proc`目录，[代码](https://elixir.bootlin.com/linux/v4.14.119/source/init/main.c#L513)

```CPP
asmlinkage void __init start_kernel(void)
{
	//...
    proc_root_init();
	//...
}
```

//TODO

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