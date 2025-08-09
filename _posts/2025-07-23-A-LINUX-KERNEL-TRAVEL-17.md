---
layout:     post
title:  Linux 内核之旅（十七）：Linux Namespace
subtitle:   PidNamespace And MntNamespace
date:       2025-07-23
author:     pandaychen
header-img:
catalog: true
tags:
    - Linux
    - Kernel
---


##  0x00    前言
命名空间用来实现内核对资源进行隔离，本文基于4.11.6的源码分析下Mnt Namespace的若干细节

##  0x01    一个容器的case
两个问题：
1.	容器内的进程pid的分配过程
2.	容器内挂载树（mount tree）的生成过程

##	0x01	pidnamespace：容器内进程pid的创建
[前文](https://pandaychen.github.io/2024/10/02/A-LINUX-KERNEL-TRAVEL-1/#%E6%A0%B8%E5%BF%83%E9%80%BB%E8%BE%91%E5%88%86%E9%85%8Dpid)已经学习过pidnamespace场景中pid的分配，这里先回顾下几个问题：

1.	容器进程中的 pid 是如何申请？
2.	内核如何显示容器中的进程号？
3.	容器pid与宿主机pid的申请区别？

从`copy_process`的实现可知，当在容器内创建一个进程时，本质上还是在宿主机内核调用`_do_fork`等一系列指令，最终复制出一个`task_struct`结构（并不是宿主机、容器单独各一个`task_struct`结构），并且通过`struct pid`及其柔性数组成员`upid`的扩展，使得不同level的pidnamespace拥有不同的pid


####	内核启动时：默认的pidnamespace
Linux内核启动的时候会有一套默认的命名空间`init_nsproxy`以及默认的 pidnamespace `init_pid_ns`，定义如下：

```CPP
struct task_struct {
 ...
 /* namespaces */
 struct nsproxy *nsproxy;
}

struct nsproxy init_nsproxy = {
	.count = ATOMIC_INIT(1),
	.uts_ns = &init_uts_ns,
	.ipc_ns = &init_ipc_ns,
	.mnt_ns = NULL,
	.pid_ns = &init_pid_ns,
	.net_ns = &init_net,
};

//https://elixir.bootlin.com/linux/v4.11.6/source/kernel/pid.c#L71
struct pid_namespace init_pid_ns = {
	.kref = {
	.refcount       = ATOMIC_INIT(2),
	},
	.pidmap = {
	[ 0 ... PIDMAP_ENTRIES-1] = { ATOMIC_INIT(BITS_PER_PAGE), NULL }
	},
	.last_pid = 0,
	.level = 0,
	.child_reaper = &init_task,
	.user_ns = &init_user_ns,
	.proc_inum = PROC_PID_INIT_INO,
};
```

而`pid_namespace`的[定义](https://elixir.bootlin.com/linux/v4.11.6/source/include/linux/pid_namespace.h#L30)如下：

-	`level`：当前 pid 命名空间的层级，默认命名空间的 level 初始化是 `0`（根节点）。这是一个表示树的层次结构的节点（`parent`表示父节点）。如果有多个命名空间构成一棵树结构
-	`pidmap`：用于标识pid分配的bitmap（位置`1`表示被分配出去）

```CPP

struct pid_namespace {
	struct kref kref;
	struct pidmap pidmap[PIDMAP_ENTRIES];
	struct rcu_head rcu;
	int last_pid;
	unsigned int nr_hashed;
	struct task_struct *child_reaper;
	struct kmem_cache *pid_cachep;
	unsigned int level;
	struct pid_namespace *parent;
#ifdef CONFIG_PROC_FS		//for procfs
	struct vfsmount *proc_mnt;
	struct dentry *proc_self;
	struct dentry *proc_thread_self;
#endif
	struct user_namespace *user_ns;
	struct ucounts *ucounts;
	struct work_struct proc_work;
	kgid_t pid_gid;
	int hide_pid;
	int reboot;	/* group exit code if this pidns was rebooted */
	struct ns_common ns;
};
```

`INIT_TASK` 即`0`号进程（idle 进程），默认绑定`init_nsproxy`，若创建进程时不指定命名空间，所有进程使用的都是使用默认命名空间

```CPP
#define INIT_TASK(tsk) \
{ 
	.state  = 0,      \
	.stack  = &init_thread_info,    \
	.usage  = ATOMIC_INIT(2),    \
	.flags  = PF_KTHREAD,     \
	.prio  = MAX_PRIO-20,     \
	.static_prio = MAX_PRIO-20,     \
	.normal_prio = MAX_PRIO-20,     \
	...
	.nsproxy = &init_nsproxy,    \
	......
}
```

####	Docker容器下的进程创建
参考[Linux 容器底层工作机制：从 500 行 C 代码到生产级容器运行时](https://arthurchiao.art/blog/linux-container-and-runtime-zh/#25-clone-%E5%90%AF%E5%8A%A8%E5%AD%90%E8%BF%9B%E7%A8%8B%E8%BF%90%E8%A1%8C%E5%AE%B9%E5%99%A8)

创建Docker容器进程(或者在容器中启动进程)时，通常要指定如下选项。其中指定了 `CLONE_NEWPID` 要创建一个独立的 pid 命名空间出来

```CPP
int flags = CLONE_NEWNS | CLONE_NEWCGROUP | CLONE_NEWPID | CLONE_NEWIPC | CLONE_NEWNET | CLONE_NEWUTS;
child_pid = clone(child, stack + STACK_SIZE, flags | SIGCHLD, &config);
```

主要涉及的内核函数如下：

```BASH
copy_process
	|-  copy_namespaces
	|-  alloc_pid
	|-  attach_pid
```


####	独立挂载
如下命令，这样在容器启动后，容器内会自动创建/soft的目录。通过这种方式，我们可以明确一点，即-v参数中，冒号":"前面的目录是宿主机目录，后面的目录是容器内目录

```BASH
docker run -it -v /test:/soft centos /bin/bash
```

##  0x01 Mnt namespace   

####    数据结构


```CPP
//https://elixir.bootlin.com/linux/v4.11.6/source/fs/mount.h#L7
struct mnt_namespace {
	atomic_t		count;
	struct ns_common	ns;
	struct mount *	root;           //指向本namespace中根目录 / 指向的mount结构
	struct list_head	list;
	struct user_namespace	*user_ns;
	struct ucounts		*ucounts;
	u64			seq;	/* Sequence number to prevent loops */
	wait_queue_head_t poll;
	u64 event;
	unsigned int		mounts; /* # of mounts in the namespace */
	unsigned int		pending_mounts;
};
```

![mnt_namespace]()

##  0x0 参考
-   [Linux 容器底层工作机制：从 500 行 C 代码到生产级容器运行时（2023）](https://arthurchiao.art/blog/linux-container-and-runtime-zh/)