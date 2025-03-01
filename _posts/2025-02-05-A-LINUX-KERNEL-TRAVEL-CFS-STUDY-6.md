---
layout:     post
title:  Linux 内核之旅（六）：进程调度（CFS）
subtitle:   
date:       2025-01-05
author:     pandaychen
header-img:
catalog: true
tags:
    - Linux
---

##  0x00    前言
本文学习下CFS调度算法（Completely Fair Scheduler，完全公平调度器）用于Linux系统中普通进程的调度

-	调度实体：对应结构`sched_entity`
-	红黑树：CFS采用了红黑树算法来管理所有的调度实体，算法效率`O(log(n))`
-	vruntime：CFS跟踪调度实体sched_entity的虚拟运行时间vruntime，平等对待运行队列中的调度实体sched_entity，将执行时间少的调度实体sched_entity排列到红黑树的左边

##  0x01    CFS数据结构及关系
-	`task_struct`：每一个调度类并不是直接管理task_struct，而是关联调度实体
-	`sched_entity`：调度实体
-	`cfs_rq`	

根据上文的介绍，了解到CFS调度器使用`sched_entity`跟踪调度信息。CFS调度器使用`cfs_rq`跟踪就绪队列信息以及管理就绪态调度实体，并维护一棵按照虚拟时间排序的红黑树，`tasks_timeline->rb_root`是红黑树的根节点，`tasks_timeline->rb_leftmost`指向红黑树中最左边的调度实体节点（虚拟时间最小的调度实体），为了更快的选择最适合运行的调度实体，`rb_leftmost`相当于一个缓存。每个就绪态的调度实体`sched_entity`包含插入红黑树中使用的节点`rb_node`，同时`vruntime`成员记录已经运行的`task_struct`虚拟时间。CFS算法会选择红黑树最左边的进程运行，随着系统时间的推移，原来左边运行过的进程慢慢的会移动到红黑树的右边，原来右边的进程也会最终跑到最左边。它们之间的关系如下图：

![cfs_relation](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/scheduler/cfs_process_schedule_impl.jpeg)

####	虚拟运行时间vruntime
![]()

####	普通进程的优先级

##	0x02	CFS的运行原理

####    进程创建
进程的创建是通过`do_fork()`完成，调用链如下：`do_fork()`----> `_do_fork()` ----> `copy_process()`----> `sched_fork()`；当fork 进程时，核心`copy_process`以复制父进程的方式来生成一个新的`task_struct`，然后调用`wake_up_new_task` 函数将新进程添加到CPU的就绪队列中，等待调度器调度执行

```CPP
long _do_fork(unsigned long clone_flags,unsigned long stack_start,unsigned long stack_size,int __user *parent_tidptr,int __user *child_tidptr,unsigned long tls){
    struct task_struct *p;
    int trace = 0;
    long nr;
    //......
    // 复制结构
    p = copy_process(clone_flags, stack_start, stack_size,child_tidptr, NULL, trace, tls, NUMA_NO_NODE);
    //......
    if (!IS_ERR(p)) {
        struct pid *pid;
        pid = get_task_pid(p, PIDTYPE_PID);
        nr = pid_vnr(pid);
        if (clone_flags & CLONE_PARENT_SETTID)
            put_user(nr, parent_tidptr);
        //......
        // 唤醒新进程
        wake_up_new_task(p);
        //......
        put_pid(pid);
    }
    //...
} 


static struct task_struct *copy_process(...){
    // 复制进程task_struct结构体
    struct task_struct *p;
    p = dup_task_struct(current, ...);
    // 复制files_struct
    retval = copy_files(clone_flags,p)
    // 复制fs_struct
    retval = copy_fs(clone_flags,p)
    // 复制mm_struct
    retval = copy_mm(clone_flags,p)
    // 复制进程的命名空间 nsproxy
    retval = copy_namespace(clone_flags,p)
    // 申请pid并设置进程号
    pid = alloc_pid(p->nsproxy->pid_ns_for_children,...);
    p->pid = pid_nr(pid);
    if (clone_flags & CLONE_THREAD) {
        p->tgid = current->tgid;
    }else{
        p->tgid = p->pid;
    }
    ...
}
```

一个新创建的普通进程，从调度的代码视角看，`copy_process`函数对新进程的`task_struct` 进行各种初始化，其中会调用`sched_fork` 来完成调度相关的初始化，核心逻辑如下：

```CPP
static struct task_struct *copy_process(...){
    //...
    retval = sched_fork(clone_flags, p);

	//...
}

// sched_fork方法主要做fork相关的操作。传递的参数p就是创建的task_struct。
int sched_fork(unsigned long clone_flags, struct task_struct *p){
    __schedd_fork(clone_flags, p);
    p->__state = TASK_NEW;
    if (rt_prio(p->prio))
        p->sched_class = &rt_sched_class;
    else
        p->sched_class = &fair_sched_class; // fair_sched_class 是一个全局对象，为CFS调度算法的实现
}

static void __sched_fork(struct task_struct *p){
    p ->on_rq = 0;
    //...
    p->se.nr_migrations = 0;
    p->se.vruntime = 0; // 注意：新进程是0对老进程不公平，在新进程真正被加入运行队列时，会将其值设置为cfs_rq->min_vruntime
}

void wake_up_new_task(struct task_struct *p){
    // 为进程选择一个合适的cpu
    cpu = select_task_rq(p, task_cpu(p)
    // 为进程指定运行队列
    __set_task_cpu(p,cpu, WF_FORK));
    // 将进程添加到运行队列红黑树
    rq = __task_rq_lock(p);
    activate_task(rq, p, 0);
}
```

需要说明的是上面`wake_up_new_task`实现中，通过调用`select_task_rq()`函数重新选择CPU（逻辑核），通过调用调度类中`select_task_rq`方法选择调度类中最空闲的CPU。如此在缓存性能和空闲核两个点做权衡，同等条件会尽量优先考虑缓存命中率，选择同`L1`/`L2`的核，其次会选择同一个物理CPU上的（共享`L3`），最坏情况下去选择另一个（负载最小的）物理CPU上的核，称之为漂移。通常由于宿主机的CPU利用率过高的水平导致出现了漂移的情况，进程在不同的核上运行概率增加，导致缓存MISS，穿透到内存的访问次数增加，进程的运行性能就会下降


##  0x0  参考
-   [【原创】（五）Linux进程调度-CFS调度器](https://www.cnblogs.com/LoyenWang/p/12495319.html)
-   [CFS调度器（1）-基本原理](http://www.wowotech.net/process_management/447.html)
-   [调度系统设计精要](https://mp.weixin.qq.com/s/R3BZpYJrBPBI0DwbJYB0YA)
-   [一文搞懂linux cfs调度器](https://zhuanlan.zhihu.com/p/556295381)