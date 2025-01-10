---
layout:     post
title:  Linux 内核之旅（三）：虚拟内存管理
subtitle:
date:       2024-11-05
author:     pandaychen
header-img:
catalog: true
tags:
    - Linux
---

##  0x00    前言
前文讨论了进程，正在执行的程序，是可执行程序的动态实例，它是一个承担分配系统资源的实体，但操作系统创建进程时，会为进程创建相应的内存空间，这个内存空间称为进程的地址空间，每一个进程的地址空间都是独立的；当一个进程有了进程的地址空间，那么其管理结构被称为内存描述符`mm_struct`

##  0x01    内存描述符：mm_struct

```CPP
struct mm_struct {
//mmap指向虚拟区间链表
    struct vm_area_struct * mmap;       /* list of VMAs */
//指向红黑树
    struct rb_root mm_rb;
//指向最近的虚拟空间
    struct vm_area_struct * mmap_cache; /* last find_vma result */
//
    unsigned long (*get_unmapped_area) (struct file *filp,
                unsigned long addr, unsigned long len,
                unsigned long pgoff, unsigned long flags);
    void (*unmap_area) (struct mm_struct *mm, unsigned long addr);
    unsigned long mmap_base;        /* base of mmap area */
    unsigned long task_size;        /* size of task vm space */
    unsigned long cached_hole_size;     /* if non-zero, the largest hole below free_area_cache */
    unsigned long free_area_cache;      /* first hole of size cached_hole_size or larger */
//指向进程的页目录
    pgd_t * pgd;
//空间中有多少用户
    atomic_t mm_users;          /* How many users with user space? */
//引用计数；描述有多少指针指向当前的mm_struct
    atomic_t mm_count;          /* How many references to "struct mm_struct" (users count as 1) */
//虚拟区间的个数
    int map_count;              /* number of VMAs */
    struct rw_semaphore mmap_sem;
//保护任务页表
    spinlock_t page_table_lock;     /* Protects page tables and some counters */
//所有mm的链表
    struct list_head mmlist;        /* List of maybe swapped mm's.  These are globally strung
                         * together off init_mm.mmlist, and are protected
                         * by mmlist_lock
                         */

    /* Special counters, in some configurations protected by the
     * page_table_lock, in other configurations by being atomic.
     */
    mm_counter_t _file_rss;
    mm_counter_t _anon_rss;

    unsigned long hiwater_rss;  /* High-watermark of RSS usage */
    unsigned long hiwater_vm;   /* High-water virtual memory usage */

    unsigned long total_vm, locked_vm, shared_vm, exec_vm;
    unsigned long stack_vm, reserved_vm, def_flags, nr_ptes;
//start_code:代码段的起始地址
//end_code:代码段的结束地址
//start_data:数据段起始地址
//end_data:数据段结束地址
    unsigned long start_code, end_code, start_data, end_data;
//start_brk:堆的起始地址
//brk:堆的结束地址
//start_stack:栈的起始地址
    unsigned long start_brk, brk, start_stack;
//arg_start,arg_end:参数段的起始和结束地址
//env_start,env_end:环境段的起始和结束地址
    unsigned long arg_start, arg_end, env_start, env_end;

    unsigned long saved_auxv[AT_VECTOR_SIZE]; /* for /proc/PID/auxv */

    struct linux_binfmt *binfmt;

    cpumask_t cpu_vm_mask;

    /* Architecture-specific MM context */
    mm_context_t context;

    /* Swap token stuff */
    /*
     * Last value of global fault stamp as seen by this process.
     * In other words, this value gives an indication of how long
     * it has been since this task got the token.
     * Look at mm/thrash.c
     */
    unsigned int faultstamp;
    unsigned int token_priority;
    unsigned int last_interval;

    unsigned long flags; /* Must use atomic bitops to access the bits */

    struct core_state *core_state; /* coredumping support */
#ifdef CONFIG_AIO
    spinlock_t      ioctx_lock;
    struct hlist_head   ioctx_list;
#endif
#ifdef CONFIG_MM_OWNER
    /*
     * "owner" points to a task that is regarded as the canonical
     * user/owner of this mm. All of the following must be true in
     * order for it to be changed:
     *
     * current == mm->owner
     * current->mm != mm
     * new_owner->mm == mm
     * new_owner->alloc_lock is held
     */
    struct task_struct *owner;
#endif

#ifdef CONFIG_PROC_FS
    /* store ref to file /proc/<pid>/exe symlink points to */
    struct file *exe_file;
    unsigned long num_exe_file_vmas;
#endif
#ifdef CONFIG_MMU_NOTIFIER
    struct mmu_notifier_mm *mmu_notifier_mm;
#endif
};
```


##  0x0  参考
-   [4.6 深入理解 Linux 虚拟内存管理](https://www.xiaolincoding.com/os/3_memory/linux_mem.html)
-   [4.7 深入理解 Linux 物理内存管理](https://www.xiaolincoding.com/os/3_memory/linux_mem2.html#_6-1-%E5%8C%BF%E5%90%8D%E9%A1%B5%E7%9A%84%E5%8F%8D%E5%90%91%E6%98%A0%E5%B0%84)
