---
layout:     post
title:  Linux 内核之旅（三）：虚拟内存管理（上）
subtitle:   进程视角的虚拟内存管理
date:       2024-11-05
author:     pandaychen
header-img:
catalog: true
tags:
    - Linux
    - Kernel
---

##  0x00    前言
前文讨论了进程，正在执行的程序，是可执行程序的动态实例，它是一个承担分配系统资源的实体，但操作系统创建进程时，会为进程创建相应的内存空间，这个内存空间称为进程的地址空间，**每一个进程的地址空间（虚拟内存空间）都是独立的**；当一个进程有了进程的地址空间，那么其管理结构被称为内存描述符`mm_struct`。有趣的说，虚拟内存其实是 CPU 和操作系统使用的一个障眼法，联手给进程编织了一个假象，让进程误以为自己独占了全部的内存空间
首先明确一点是进程的虚拟地址空间有两部分组成：内核空间和用户空间，内核空间各个进程直接共享，而用户空间彼此隔离

本文就来学习下**进程虚拟内存空间在内核中的布局以及管理**，内核版本基于[v4.11.6](https://elixir.bootlin.com/linux/v4.11.6/source/include) 版本

![linux_memory_management.png]()

####    虚拟内存地址

####    虚拟内存空间分布（32位）

![32-vm]()

`32`位机器上进程的用户空间大小为 `3GB`。内核将一个进程的用户空间划分为多个段：
-   代码段：用于存放程序中可执行代码的段
-   数据段：用于存放已经初始化的全局变量或静态变量的段（如 `int global = 10;` 定义的全局变量）
-   未初始化数据段：用于存放未初始化的全局变量或静态变量的段（如`int global;` 定义的全局变量）
-   堆：用于存放使用 `malloc` 函数申请的内存
-   `mmap`区：用于存放使用 `mmap` 函数映射的内存区（文件映射与匿名映射区），详见后文描述
-   栈：用于存放函数局部变量和函数参数

####    虚拟内存空间分布（64位）

![64-vm]()


####    页表
参考[一步一图带你构建 Linux 页表体系：详解虚拟内存如何与物理内存进行映射](https://mp.weixin.qq.com/s?__biz=Mzg2MzU3Mjc3Ng==&mid=2247488477&idx=1&sn=f8531b3220ea3a9ca2a0fdc2fd9dabc6&chksm=ce77d59af9005c8c2ef35c7e45f45cbfc527bfc4b99bbd02dbaaa964d174a4009897dd329a4d&scene=178&cur_album_id=2559805446807928833#rd)

##  0x01    进程虚拟内存空间管理
无论是在 `32` 位机器上还是在 `64` 位机器上，进程虚拟内存空间的核心区域分布的相对位置是一样的，这个结构是怎么在内核反映的？

![]()

####    进程虚拟内存描述符：mm_struct
已知每个内核进程描述符结构体`task_struct`中嵌套了一个`mm_struct`结构体指针，该结构体中包含了的该进程虚拟内存空间的全部信息。同时每个进程都有唯一的 `mm_struct` 结构体，这说明每个进程的虚拟地址空间都是独立，互不干扰的。当调用 `fork()` 函数创建进程的时候，表示进程地址空间的 `mm_struct` 结构会随着进程描述符 `task_struct` 的创建而创建

创建`mm_struct`的逻辑在内核函数[`copy_mm`](https://elixir.bootlin.com/linux/v4.11.6/source/kernel/fork.c#L1174)中，注意这里区分用户态进程和用户态线程，前者会单独复制出一份`mm_struct`，后者是共享父进程的`mm_struct`。`copy_mm` 函数首先会将父进程的虚拟内存空间 `current->mm` 赋值给指针 `oldmm`，然后通过 `dup_mm` 函数将父进程的虚拟内存空间以及相关页表拷贝到子进程的 `mm_struct` 结构中，最后将拷贝出来的 `mm_struct` 赋值给子进程的 `task_struct` 结构，最终布局[参考](https://pandaychen.github.io/2024/10/02/A-LINUX-KERNEL-TRAVEL-1/#%E8%BF%9B%E7%A8%8B%E4%B8%8E%E7%BA%BF%E7%A8%8B)

```CPP
struct task_struct {
    //.......
    // 内存描述符表示进程虚拟地址空间
    struct mm_struct *mm;
    //.......
}
struct mm_struct {
          struct vm_area_struct * mmap;       /* list of VMAs */
          struct rb_root mm_rb;     /*红黑树的根节点*/
          struct vm_area_struct * mmap_cache;      /* last find_vma result */
        //.......
}
```

`mm_struct`结构体定义如下：

```CPP
struct mm_struct {
    //mmap指向虚拟区间链表
    struct vm_area_struct * mmap;       /* list of VMAs */
    //指向红黑树的根节点
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

####    task_size
`task_size`：定义了用户态地址空间与内核态地址空间之间的分界线，如`64` 位系统中用户地址空间和内核地址空间的分界线在 `0x0000 7FFF FFFF F000` 地址处，那么`task_struct.mm_struct` 结构中的 `task_size` 为 `0x0000 7FFF FFFF F000`

![]()

####    进程虚拟内存空间布局（区间端点）

```CPP
struct mm_struct {
    unsigned long task_size;    /* size of task vm space */
    unsigned long start_code, end_code, start_data, end_data;
    unsigned long start_brk, brk, start_stack;
    unsigned long arg_start, arg_end, env_start, env_end;
    unsigned long mmap_base;  /* base of mmap area */
    unsigned long total_vm;    /* Total pages mapped */
    unsigned long locked_vm;  /* Pages that have PG_mlocked set */
    unsigned long pinned_vm;  /* Refcount permanently increased */
    unsigned long data_vm;    /* VM_WRITE & ~VM_SHARED & ~VM_STACK */
    unsigned long exec_vm;    /* VM_EXEC & ~VM_WRITE & ~VM_STACK */
    unsigned long stack_vm;    /* VM_STACK */

    //......
}
```

![mm_struct]()

####    vm_area_struct（VMA）
Linux 内核按照功能上的差异，把虚拟内存空间划分为多个段。那么在内核中，是通过`vm_area_struct`结构来管理这些段，如代码段、数据段、BSS 段、堆、栈以及文件映射与匿名映射区等，在内核中都是 `struct vm_area_struct` 结构来表示的

内核使用结构体 [`vm_area_struct`](https://elixir.bootlin.com/linux/v4.11.6/source/include/linux/mm_types.h#L284)来**描述用户虚拟内存空间的各个逻辑区域：代码段，数据段，堆，内存映射区，栈等，也称为VMA（virtual memory area）**

每个 `vm_area_struct` 结构对应于虚拟内存空间中的唯一虚拟内存区域 VMA，其中`vm_start` 指向了这块虚拟内存区域的起始地址（最低地址），`vm_start` 本身包含在这块虚拟内存区域内，`vm_end` 指向了这块虚拟内存区域的结束地址（最高地址），而 `vm_end` 本身包含在这块虚拟内存区域之外，所以 `vm_area_struct` 结构描述的是 `[vm_start，vm_end)` 这样一段左闭右开的虚拟内存区域

```CPP
//内核通过 vm_area_struct 结构（虚拟内存区）来管理各个段
struct vm_area_struct {
    struct mm_struct *vm_mm; /* The address space we belong to. */
	unsigned long vm_start;		/* Our start address within vm_mm. */
	unsigned long vm_end;		/* The first byte after our end address
					   within vm_mm. */

    /* linked list of VM areas per task, sorted by address */
	struct vm_area_struct *vm_next, *vm_prev;
	/*
	 * Access permissions of this VMA.
	 */
	pgprot_t vm_page_prot;
	unsigned long vm_flags;	

    //每个 VMA 区域都是红黑树中的一个节点，通过 struct vm_area_struct 结构中的 vm_rb 将自己连接到红黑树中
    struct rb_node vm_rb;

	struct anon_vma *anon_vma;	/* Serialized by page_table_lock */
    struct file * vm_file;		/* File we map to (can be NULL). */
	unsigned long vm_pgoff;		/* Offset (within vm_file) in PAGE_SIZE
					   units */	
	void * vm_private_data;		/* was vm_pte (shared mem) */
	/* Function pointers to deal with this struct. */
	const struct vm_operations_struct *vm_ops;
}
```

-   `vm_mm`：指向进程的内存管理对象，每个进程都有一个类型为 `mm_struct` 的内存管理对象，用于管理进程的虚拟内存空间和内存映射等
-   `vm_start`：虚拟内存区的起始虚拟内存地址
-   `vm_end`：虚拟内存区的结束虚拟内存地址
-   `vm_next`：Linux 会通过链表把进程的所有虚拟内存区连接起来，这个字段用于指向下一个虚拟内存区
-   `vm_page_prot`：主要用于保存当前虚拟内存区所映射的物理内存页的读写权限
-   `vm_flags`：标识当前虚拟内存区的功能特性
-   `vm_rb`：某些场景中需要通过虚拟内存地址查找对应的虚拟内存区，为了加速查找过程，内核以虚拟内存地址作为key，把进程所有的虚拟内存区保存到一棵红黑树中，而这个字段就是红黑树的节点结构
-   `vm_ops`：每个虚拟内存区都可以自定义一套操作接口，通过操作接口，能够让虚拟内存区实现一些特定的功能，比如：把虚拟内存区映射到文件。而 `vm_ops` 字段就是虚拟内存区的操作接口集，一般在创建虚拟内存区时指定

内核通过一个链表和一棵红黑树来管理进程中所有的段。`mm_struct` 结构的 `mmap` 字段就是链表的头节点，而 `mm_rb` 字段就是红黑树的根节点，如下：

![vm_area_struct](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/memory/vm-area-struct-layout.png)

####    mm_rb && mmap

```CPP
struct mm_struct {
     struct rb_root mm_rb;              //红黑树的root

     //vm_area_struct 链表首节点
     struct vm_area_struct *mmap;		/* list of VMAs */
}
```

几个要点：

-   `mmap`：串联起了整个虚拟内存空间中的虚拟内存区域，即一个双向链表将虚拟内存空间中的这些虚拟内存区域 VMA 串联起来。`vm_area_struct` 结构中的 `vm_next`/`vm_prev` 指针分别指向 VMA 节点所在双向链表中的后继和前驱节点，内核中的这个 VMA 双向链表是有顺序的，所有 VMA 节点按照低地址到高地址的增长方向排序。此外，双向链表中的最后一个 VMA 节点的 `vm_next` 指向 `NULL`，双向链表的头指针存储在内存描述符 `struct mm_struct` 结构中的 `mmap`成员
-   在每个虚拟内存区域 VMA 中又通过 `struct vm_area_struct` 中的 `vm_mm` 指针指向了所属的虚拟内存空间 `mm_struct`
-   可通过 `cat /proc/pid/maps`/`pmap pid` 查看进程的虚拟内存空间布局以及其中包含的所有内存区域（其实现原理就是通过遍历内核中的这个 `vm_area_struct` 双向链表获取）
-   `mm_rb`：红黑树的root节点，每个`vm_area_struct`都包含了`struct rb_node vm_rb`成员即为红黑树的节点
-   在内核中，**同样的内存区域 `vm_area_struct` 会有两种组织形式，一种是双向链表用于高效的遍历，另一种就是红黑树用于高效的查找**

使用rbtree组织VMA区域的场景如下：
-   需要根据特定虚拟内存地址在虚拟内存空间中查找特定的VMA虚拟内存区域，尤其在进程虚拟内存空间中包含的内存区域 VMA 比较多的情况下，使用rbtree查找更为高效

![final](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/memory/mm_struct_all_view.jpg)

####    小结
单个进程虚拟内存空间中的所有 VMA 在内核中有两种组织形式：

-   双向链表：用于高效的遍历进程 VMA，这个 VMA 双向链表是有顺序的，所有 VMA 节点在双向链表中的排列顺序是按照虚拟内存低地址到高地址进行的
-   红黑树：用于在进程空间中高效的查找 VMA，因为在进程虚拟内存空间中不仅仅是只有代码段，数据段，BSS 段，堆，栈这些虚拟内存区域 VMA，尤其是在数据密集型应用进程中，文件映射与匿名映射区里也会包含有大量的 VMA，进程的各种动态链接库所映射的虚拟内存在这里，进程运行过程中进行的匿名映射，文件映射所需要的虚拟内存也在这里

```CPP
// 进程虚拟内存空间描述符
struct mm_struct {
    // 串联组织进程空间中所有的 VMA  的双向链表 
    struct vm_area_struct *mmap;  /* list of VMAs */
    // 管理进程空间中所有 VMA 的红黑树
    struct rb_root mm_rb;
}

// 虚拟内存区域描述符
struct vm_area_struct {
    // vma 在 mm_struct->mmap 双向链表中的前驱节点和后继节点
    struct vm_area_struct *vm_next, *vm_prev;
    // vma 在 mm_struct->mm_rb 红黑树中的节点
    struct rb_node vm_rb;
    unsigned long vm_start;     /* Our start address within vm_mm. */
    unsigned long vm_end;       /* The first byte after our end address */
    struct file * vm_file;      /* File we map to (can be NULL). */
    unsigned long vm_pgoff;     /* Offset (within vm_file) in PAGE_SIZE */
}
```

![mm_struct-vm_area_struct-ds2]()

##  0x02    进程虚拟内存管理：ELF加载
本小节主要讨论下进程的虚拟内存区是如何建立起来的，主要涉及到：
-   ELF文件

####    ELF：Executable and Linkable Format
在 Linux 系统中使用ELF格式来存储一个可执行的应用程序，一个 ELF 文件由以下三部分组成：

![elf]()

-   ELF 头（ELF header）：描述应用程序的类型、CPU架构、入口地址、程序头表偏移和节头表偏移等
-   程序头表（Program header table）：列举了所有有效的段（segments）和其属性，程序头表需要加载器将文件中的段加载到虚拟内存段中
-   节头表（Section header table）：包含对节（sections）的描述

当内核加载一个应用程序时，就是通过读取 ELF 文件的信息，然后把文件中所有的段加载到虚拟内存的段中。ELF 文件通过程序头表`elf64_phdr`来描述应用程序中所有的段，表中的每一个项都描述一个段的信息，程序加载器可以通过 ELF 头中获取到程序头表的偏移量，然后通过程序头表的偏移量读取到程序头表的数据，再通过程序头表来获取到所有段的信息

```CPP
typedef struct elf64_phdr {
    Elf64_Word p_type;     // 段的类型
    Elf64_Word p_flags;    // 可读写标志
    Elf64_Off p_offset;    // 段在ELF文件中的偏移量
    Elf64_Addr p_vaddr;    // 段的虚拟内存地址
    Elf64_Addr p_paddr;    // 段的物理内存地址
    Elf64_Xword p_filesz;  // 段占用文件的大小
    Elf64_Xword p_memsz;   // 段占用内存的大小
    Elf64_Xword p_align;   // 内存对齐
} Elf64_Phdr;
```

####    ELF的加载过程
要加载一个程序，需要调用 `execve` 系统调用来完成， `execve` 系统调用的调用栈如下，`execve` 系统调用最终会调用 `load_elf_binary` 函数来加载程序的 ELF 文件，接下来简单描述下`load_elf_binary`的过程

```BASH
sys_execve
 ->do_execve
    ->do_execveat_common
       -> __do_execve_file
           -> exec_binprm
              -> search_binary_handler
                 -> load_elf_binary
```

####    load_elf_binary的实现

1、读取并检查ELF头

```CPP
//这段代码的逻辑主要是读取应用程序的 ELF 头，然后检查 ELF 头信息是否合法
static int load_elf_binary(struct linux_binprm *bprm, struct pt_regs *regs)
{
    //...
    struct {
        struct elfhdr elf_ex;
        struct elfhdr interp_elf_ex;
    } *loc;

    loc = kmalloc(sizeof(*loc), GFP_KERNEL);
    if (!loc) {
        retval = -ENOMEM;
        goto out_ret;
    }

    // 1. 获取ELF头
    loc->elf_ex = *((struct elfhdr *)bprm->buf);

    retval = -ENOEXEC;
    // 2. 检查ELF签名是否正确
    if (memcmp(loc->elf_ex.e_ident, ELFMAG, SELFMAG) != 0)
        goto out;

    // 3. 是否是可执行文件或者动态库
    if (loc->elf_ex.e_type != ET_EXEC && loc->elf_ex.e_type != ET_DYN)
        goto out;

    // 4. 检查系统架构是否正确
    if (!elf_check_arch(&loc->elf_ex))
        goto out;
    //...
}
```

2、读取程序头表，主要包含如下流程：

-   从 ELF 头的信息中获取到程序头表的大小
-   调用 `kmalloc` 函数申请一块内存来保存程序头表
-   调用 `kernel_read` 函数从 ELF 文件中读取程序头表的数据，保存到 `elf_phdata` 变量中，程序头表的偏移量可以通过 ELF 头的 `e_phoff` 字段获取

```CPP
static int load_elf_binary(struct linux_binprm *bprm, struct pt_regs *regs)
{
    // ......
    // ......
    // 接上面的代码
    size = loc->elf_ex.e_phnum * sizeof(struct elf_phdr); // 程序头表的大小
    retval = -ENOMEM;

    elf_phdata = kmalloc(size, GFP_KERNEL); // 申请一块内存来保存程序头表
    if (!elf_phdata)
        goto out;

	// 从ELF文件中读取程序头表的数据, 并且保存到 elf_phdata 变量中
    retval = kernel_read(bprm->file, loc->elf_ex.e_phoff, (char *)elf_phdata, size);
    if (retval != size) {
        if (retval >= 0)
            retval = -EIO;
        goto out_free_ph;
    }
    //...
}
```

3、加载段到虚拟内存，把段加载到虚拟内存主要通过 `elf_map` 函数完成

-   遍历程序头表所有的段
-   判断段是否需要加载
-   获取段的可读写权限和段的虚拟内存地址
-   调用 `elf_map` 函数把段加载到虚拟内存

```CPP
    // 遍历程序头表所有的段
    for (i = 0, elf_ppnt = elf_phdata; i < loc->elf_ex.e_phnum; i++, elf_ppnt++) {
        int elf_prot = 0, elf_flags;
        unsigned long k, vaddr;

        if (elf_ppnt->p_type != PT_LOAD)  // 判断段是否需要加载
            continue;
        //...
        // 段的可读写权限
        if (elf_ppnt->p_flags & PF_R)
            elf_prot |= PROT_READ;
        if (elf_ppnt->p_flags & PF_W)
            elf_prot |= PROT_WRITE;
        if (elf_ppnt->p_flags & PF_X)
            elf_prot |= PROT_EXEC;

        elf_flags = MAP_PRIVATE | MAP_DENYWRITE | MAP_EXECUTABLE;

        vaddr = elf_ppnt->p_vaddr;  // 获取段的虚拟内存地址
        //...
        // 把段加载到虚拟内存
        error = elf_map(bprm->file, load_bias + vaddr, elf_ppnt, elf_prot, elf_flags, 0);
        //...
    }
```

4、 `elf_map` 函数的调用栈及核心流程，`elf_map` 函数最终会调用 `mmap_region` 来完成加载段到虚拟内存

```BASH
elf_map
 |--> do_mmap
   |--> do_mmap_pgoff
      |--> mmap_region
```

`mmap_region`的主要流程如下：

-   调用 `kmem_cache_zalloc` 函数申请一个 `vm_area_struct`（虚拟内存区）结构
-   设置 `vm_area_struct` 结构各个字段的值
-   调用 `vma_link` 函数把 `vm_area_struct` 结构连接到虚拟内存区链表和红黑树中，`vma_link`[实现](https://elixir.bootlin.com/linux/v4.11.6/source/mm/mmap.c#L589)
-   通过上面的过程，内核就把应用程序的所有段加载到虚拟内存中

```CPP
unsigned long 
mmap_region(struct file *file, unsigned long addr, unsigned long len, 
            unsigned long flags, unsigned int vm_flags, unsigned long pgoff)
{
    struct mm_struct *mm = current->mm;
    struct vm_area_struct *vma, *prev;
    //...
    // 申请一个 vm_area_struct 结构
    vma = kmem_cache_zalloc(vm_area_cachep, GFP_KERNEL);
    if (!vma) {
        error = -ENOMEM;
        goto unacct_error;
    }

    // 设置 vm_area_struct 结构各个字段的值
    vma->vm_mm = mm;
    vma->vm_start = addr;        // 段的开始虚拟内存地址
    vma->vm_end = addr + len;    // 段的结束虚拟内存地址
    vma->vm_flags = vm_flags;    // 段的功能特性
    vma->vm_page_prot = vm_get_page_prot(vm_flags);
    vma->vm_pgoff = pgoff;

    //...
    // 把 vm_area_struct 结构链接到虚拟内存区链表和红黑树中
    vma_link(mm, vma, prev, rb_link, rb_parent);
    //...
    
    return addr;
}
```

####    ELF加载之后的内存布局
以下面代码为例：
```CPP
int main()
{
   printf("Hello, World!\n");
   int *a=malloc(1024*1024);
   memset(a,0,1024*1024);
   getchar();
   return 0;
}
```

可以通过`/proc/${pid}/maps`查看内存布局，如下：

```BASH
[root@VM-X-X-tencentos ~]# cat /proc/1115271/maps 
00400000-00401000 r--p 00000000 fd:01 1892501                            /root/main
00401000-00402000 r-xp 00001000 fd:01 1892501                            /root/main
00402000-00403000 r--p 00002000 fd:01 1892501                            /root/main
00403000-00404000 r--p 00002000 fd:01 1892501                            /root/main
00404000-00405000 rw-p 00003000 fd:01 1892501                            /root/main
0178a000-017ab000 rw-p 00000000 00:00 0                                  [heap]
7f7d46a0c000-7f7d46a32000 r--p 00000000 fd:01 26356345                   /usr/lib64/libc.so.6
7f7d46a32000-7f7d46b98000 r-xp 00026000 fd:01 26356345                   /usr/lib64/libc.so.6
7f7d46b98000-7f7d46bed000 r--p 0018c000 fd:01 26356345                   /usr/lib64/libc.so.6
7f7d46bed000-7f7d46bf1000 r--p 001e0000 fd:01 26356345                   /usr/lib64/libc.so.6
7f7d46bf1000-7f7d46bf3000 rw-p 001e4000 fd:01 26356345                   /usr/lib64/libc.so.6
7f7d46bf3000-7f7d46c00000 rw-p 00000000 00:00 0 
7f7d46c00000-7f7d46c03000 r-xp 00000000 fd:01 28659740                   /usr/lib64/libonion_security.so.1.0.19
7f7d46c03000-7f7d46d03000 ---p 00003000 fd:01 28659740                   /usr/lib64/libonion_security.so.1.0.19
7f7d46d03000-7f7d46d04000 rw-p 00003000 fd:01 28659740                   /usr/lib64/libonion_security.so.1.0.19
7f7d46d04000-7f7d46d06000 rw-p 00000000 00:00 0 
7f7d46e00000-7f7d46e03000 r-xp 00000000 fd:01 25582409                   /usr/lib64/libtdsp_security.so.1.0.25
7f7d46e03000-7f7d47002000 ---p 00003000 fd:01 25582409                   /usr/lib64/libtdsp_security.so.1.0.25
7f7d47002000-7f7d47003000 rw-p 00002000 fd:01 25582409                   /usr/lib64/libtdsp_security.so.1.0.25
7f7d47140000-7f7d47142000 rw-p 00000000 00:00 0 
7f7d47142000-7f7d47143000 r--p 00000000 fd:01 26356431                   /usr/lib64/libdl.so.2
7f7d47143000-7f7d47144000 r-xp 00001000 fd:01 26356431                   /usr/lib64/libdl.so.2
7f7d47144000-7f7d47145000 r--p 00002000 fd:01 26356431                   /usr/lib64/libdl.so.2
7f7d47145000-7f7d47146000 r--p 00002000 fd:01 26356431                   /usr/lib64/libdl.so.2
7f7d47146000-7f7d47147000 rw-p 00003000 fd:01 26356431                   /usr/lib64/libdl.so.2
7f7d4714e000-7f7d47150000 rw-p 00000000 00:00 0 
7f7d47151000-7f7d47152000 r--p 00000000 fd:01 26345834                   /usr/lib64/ld-linux-x86-64.so.2
7f7d47152000-7f7d47179000 r-xp 00001000 fd:01 26345834                   /usr/lib64/ld-linux-x86-64.so.2
7f7d47179000-7f7d47183000 r--p 00028000 fd:01 26345834                   /usr/lib64/ld-linux-x86-64.so.2
7f7d47183000-7f7d47185000 r--p 00032000 fd:01 26345834                   /usr/lib64/ld-linux-x86-64.so.2
7f7d47185000-7f7d47187000 rw-p 00034000 fd:01 26345834                   /usr/lib64/ld-linux-x86-64.so.2
7ffddfc95000-7ffddfcb6000 rw-p 00000000 00:00 0                          [stack]
7ffddfd94000-7ffddfd98000 r--p 00000000 00:00 0                          [vvar]
7ffddfd98000-7ffddfd9a000 r-xp 00000000 00:00 0                          [vdso]
ffffffffff600000-ffffffffff601000 --xp 00000000 00:00 0                  [vsyscall]
```

在Linux中，进程运行时的物理内存消耗主要通过驻留集大小（RSS）来衡量。RSS表示进程当前使用的物理内存页大小，包括代码段、数据段、堆、栈以及共享库部分。需要注意的是`malloc`分配的是虚拟内存，只有在实际访问（如 `memset`）时，操作系统才会通过页错误机制（即缺页中断）分配物理内存。代码段和共享库段通常被多个进程共享，因此从系统总体来看，物理内存消耗可能较少，但从单个进程的RSS角度，这些共享页仍然会被计入

对于上述这段程序，物理内存消耗主要来自以下部分：

-   程序代码段（text segment）：可执行文件本身的代码。本例中，代码段通常只有几KB
-   数据段（data segment）和BSS段：本例中无全局变量，所以数据段和BSS段很小（可能小于`1KB`），物理内存消耗可忽略
-   堆（heap）：本例子中，通过 `malloc`分配了`1MB`虚拟内存，并通过 `memset`访问全部内容，因此这`1MB`会完全占用物理内存（内存充裕时）；如果代码中没有调用 `memset`，那么 `malloc`分配的`1MB`可能不会立即占用物理内存（直到访问时才分配）
-   栈（stack）：用于局部变量和函数调用，消耗很小（通常几KB），物理内存可忽略
-   共享库（shared libraries）：程序使用 `printf`和 `getchar`（C标准库）。共享库的代码和数据会被映射到进程地址空间。物理内存会加载库中实际使用的部分（如I/O函数、缓冲区等），典型值为几百KB到`1MB`。这些页可能与其他进程共享，但RSS会计入进程使用的部分
-   其他开销：进程内核数据结构（如页表、任务结构）消耗少量物理内存（通常几KB）


####    vm_area_struct的分配
把上面的例子修改一下：

```CPP
int main() {
   printf("Hello, World!\n");
   int *a = malloc(1024 * 1024); // 分配1MB
   memset(a, 0, 1024 * 1024);
   int *b = malloc(1024 * 1024); // 再分配1MB
   memset(b, 0, 1024 * 1024);
   getchar();
   return 0;
}
```

虚拟内存布局如下：
```bash
[root@VM-119-175-tencentos ~]# cat /proc/1960685/maps 
00400000-00401000 r--p 00000000 fd:01 567111                             /root/main
00401000-00402000 r-xp 00001000 fd:01 567111                             /root/main
00402000-00403000 r--p 00002000 fd:01 567111                             /root/main
00403000-00404000 r--p 00002000 fd:01 567111                             /root/main
00404000-00405000 rw-p 00003000 fd:01 567111                             /root/main
00e2a000-00e4b000 rw-p 00000000 00:00 0                                  [heap]
7fde7ed15000-7fde7ef17000 rw-p 00000000 00:00 0 
7fde7ef17000-7fde7ef3d000 r--p 00000000 fd:01 396930                     /usr/lib64/libc.so.6
7fde7ef3d000-7fde7f098000 r-xp 00026000 fd:01 396930                     /usr/lib64/libc.so.6
7fde7f098000-7fde7f0ed000 r--p 00181000 fd:01 396930                     /usr/lib64/libc.so.6
7fde7f0ed000-7fde7f0f1000 r--p 001d5000 fd:01 396930                     /usr/lib64/libc.so.6
7fde7f0f1000-7fde7f0f3000 rw-p 001d9000 fd:01 396930                     /usr/lib64/libc.so.6
7fde7f0f3000-7fde7f100000 rw-p 00000000 00:00 0 
7fde7f100000-7fde7f103000 r-xp 00000000 fd:01 404268                     /usr/lib64/libonion_security.so.1.0.19
7fde7f103000-7fde7f203000 ---p 00003000 fd:01 404268                     /usr/lib64/libonion_security.so.1.0.19
7fde7f203000-7fde7f204000 rw-p 00003000 fd:01 404268                     /usr/lib64/libonion_security.so.1.0.19
7fde7f204000-7fde7f206000 rw-p 00000000 00:00 0 
7fde7f286000-7fde7f289000 rw-p 00000000 00:00 0 
7fde7f289000-7fde7f28a000 r--p 00000000 fd:01 397016                     /usr/lib64/libdl.so.2
7fde7f28a000-7fde7f28b000 r-xp 00001000 fd:01 397016                     /usr/lib64/libdl.so.2
7fde7f28b000-7fde7f28c000 r--p 00002000 fd:01 397016                     /usr/lib64/libdl.so.2
7fde7f28c000-7fde7f28d000 r--p 00002000 fd:01 397016                     /usr/lib64/libdl.so.2
7fde7f28d000-7fde7f28e000 rw-p 00003000 fd:01 397016                     /usr/lib64/libdl.so.2
7fde7f297000-7fde7f299000 rw-p 00000000 00:00 0 
7fde7f29a000-7fde7f29b000 r--p 00000000 fd:01 396776                     /usr/lib64/ld-linux-x86-64.so.2
7fde7f29b000-7fde7f2c1000 r-xp 00001000 fd:01 396776                     /usr/lib64/ld-linux-x86-64.so.2
7fde7f2c1000-7fde7f2cb000 r--p 00027000 fd:01 396776                     /usr/lib64/ld-linux-x86-64.so.2
7fde7f2cb000-7fde7f2cd000 r--p 00031000 fd:01 396776                     /usr/lib64/ld-linux-x86-64.so.2
7fde7f2cd000-7fde7f2cf000 rw-p 00033000 fd:01 396776                     /usr/lib64/ld-linux-x86-64.so.2
7ffc30041000-7ffc30062000 rw-p 00000000 00:00 0                          [stack]
7ffc300f2000-7ffc300f6000 r--p 00000000 00:00 0                          [vvar]
7ffc300f6000-7ffc300f8000 r-xp 00000000 00:00 0                          [vdso]
ffffffffff600000-ffffffffff601000 --xp 00000000 00:00 0                  [vsyscall]
```

下面结合`mm_struct`的成员及上述布局，稍微解读下上述结果：

-   **在 Linux 内核中，每个在 `/proc/pid/maps`中的行对应一个 `vm_area_struct`结构体，因此在当前机器上（运行的进程），该进程共有 `33` 个独立的虚拟内存区域，因此内核会创建 `33` 个`vm_area_struct`结构体，这些`vm_area_struct`被内核通过红黑树来实现高效查找**
-   程序固有区域：包括代码段、数据段、BSS段等（对应上述 maps 文件的前`5`行），这些是进程启动时由 `execve`系统调用映射的
-   堆区域（heap）：地址区间为 `00e2a000-00e4b000`，这是通过 `brk`系统调用管理的堆区域，用于小块内存分配
-   匿名内存映射区域：这是由 `malloc`分配大块内存（如`1MB`）时通过 `mmap`创建的，在上述 maps 文件中，有一个关键的匿名映射区域即`7fde7ed15000-7fde7ef17000 rw-p 00000000 00:00 0`，该区域大小为 `0x7fde7ef17000 - 0x7fde7ed15000 = 0x200000`字节（即`2MB`），权限为可读可写（`rw-p`），且是匿名映射（无文件备份）。这个`2MB`的虚拟内存区域对应代码中的两次`1MB`的`malloc`分配。由于这`2`次分配在虚拟地址空间中是连续的，并且具有相同的权限（`rw-p`），内核可能将它们合并为一个连续的 `vm_area_struct`区域（即一个2MB区域）
-   其他匿名映射：maps 文件中还有多个较小的匿名映射区域（如 `7fde7f0f3000-7fde7f100000`、`7fde7f204000-7fde7f206000`等），这些可能来自共享库的内部分配（如`libc.so.6`、`libdl.so.2`、`ld-linux-x86-64.so.2`等）或其它动态内存分配
-   其他等特殊区域：如栈（stack）、vvar、vdso、vsyscall 等

结合前文的描述，`vm_area_struct` 的管理方式大致如下：

```CPP
// 进程虚拟内存空间描述符
struct mm_struct {
    // 串联组织进程空间中所有的 VMA  的双向链表 
    struct vm_area_struct *mmap;  /* list of VMAs */
    // 管理进程空间中所有 VMA 的红黑树
    struct rb_root mm_rb;
}

// 虚拟内存区域描述符
struct vm_area_struct {
    // vma 在 mm_struct->mmap 双向链表中的前驱节点和后继节点
    struct vm_area_struct *vm_next, *vm_prev;
    // vma 在 mm_struct->mm_rb 红黑树中的节点
    struct rb_node vm_rb;
}
```

-   内存描述符（`mm_struct`）：每个进程有一个 `mm_struct`结构体，其中包含进程虚拟内存管理的所有信息。`mm_struct`中的 `mmap`字段指向一个链表，该链表按虚拟地址顺序连接所有 `vm_area_struct`结构体
-   红黑树：关联`mm_struct`中的 `mm_rb`字段，用于按虚拟地址快速查找 `vm_area_struct`。插入、删除和查找操作的时间复杂度 `O(logN)`
-   区域合并：当新的内存区域被映射（如通过 `mmap`）时，内核会检查它与相邻区域是否具有相同的权限和映射类型。如果满足条件，内核会合并这些区域为一个更大的 `vm_area_struct`，从而减少结构体数量
-   缺页中断处理：当进程访问虚拟内存时，如果页面不在物理内存中，会触发缺页中断。内核通过查找 `vm_area_struct`来确认访问的合法性，并分配物理页面

##  0x03  mmap的原理分析
在介绍mmap的机制之前，先认清几个概念与问题：

-   虚拟内存与物理内存的映射依赖页表机制
-   内存映射是按照物理内存页为单位进行的，内存管理中内存页主要分为匿名页与文件页，因此内存映射也分为两种：一种是虚拟内存对匿名物理内存页的映射，另一种是虚拟内存对文件页的映射（文件映射，通过内存文件映射可以将磁盘上的文件映射到内存中，这样可以通过读写内存来完成磁盘文件的读写）
-   完整的五级页表体系是如何构建出来的？（在进程被创建出来之后，内核也仅是会为进程分配一张全局页目录表 PGD（Page Global Directory）而已，此时进程虚拟内存空间中只存在一张顶级页目录表），PUD、PMD以及一级页表的构建时机是什么？

####    mmap API
```CPP
#include <sys/mman.h>
void* mmap(void* addr, size_t length, int prot, int flags, int fd, off_t offset);

// 内核文件：/arch/x86/kernel/sys_x86_64.c
SYSCALL_DEFINE6(mmap, unsigned long, addr, unsigned long, len,
		unsigned long, prot, unsigned long, flags,
		unsigned long, fd, unsigned long, off)
```

`mmap` 内存映射里**所谓的内存其实指的是虚拟内存**，常见有两种场景（对应于物理内存中的匿名页与文件页）：
-   匿名映射：在调用 `mmap` 进行匿名映射的时候（比如进行堆内存的分配），是将**进程虚拟内存空间中的某一段虚拟内存区域与物理内存中的匿名内存页进行映射**
-   文件映射：当调用 `mmap` 进行文件映射的时候，是将**进程虚拟内存空间中的某一段虚拟内存区域与磁盘中某个文件中的某段区域进行映射**

####    文件映射与匿名映射区（files and anonymous mappings）
内存映射所消耗的虚拟内存位于进程虚拟内存空间的文件映射与匿名映射区，在文件映射与匿名映射这段虚拟内存区域中，包含了若干段的虚拟映射区，每当调用一次 `mmap` 进行内存映射的时候，内核都会在文件映射与匿名映射区中划分出一段虚拟映射区出来，这段虚拟映射区就是申请到的虚拟内存

`mmap`在这段区域中申请的虚拟内存块的起始地址及大小由`mmap`的参数决定，若通过 `mmap` 映射的是磁盘上的一个文件，那么就需要通过参数 `fd` 来指定要映射文件的描述符（file descriptor），通过参数 `offset` 来指定文件映射区域在文件中偏移（由于内存页和磁盘块大小一般情况下都是 `4k`，因此这里`offset`按`4k`对齐）

![files-and-anonymous-mappings]()

####    mmap的参数&&功能
`mmap`的参数决定了申请虚拟内存的起始地址与size，`addr`/`length` 必须要按照 `PAGE_SIZE`（`4k`大小对齐）

-   `addr`：表示要映射的这段虚拟内存区域在进程虚拟内存空间中的起始地址（虚拟内存地址），一般会设置为 `NULL`，意思就是完全交由内核来帮决定虚拟映射区的起始地址
-   `length`：表示要申请的这段虚拟内存的size。如果是匿名映射，`length` 决定了要映射的匿名物理内存有多大；如果是文件映射，`length`决定了要映射的文件区域有多大

mmap常见的参数组合搭配如下：

-   私有匿名映射：`MAP_PRIVATE | MAP_ANONYMOUS` 表示私有匿名映射，常常利用这种映射方式来申请虚拟内存。比如使用 glibc的 `malloc` 函数进行虚拟内存申请时，当申请的内存大于 `128K` 的时候，`malloc` 就会调用 `mmap` 采用私有匿名映射的方式来申请堆内存。由于它是私有映射，所以申请到的内存是进程独占的，多进程之间不能共享。特别强调一下 `mmap` 私有匿名映射申请到的只是虚拟内存，内核只是在进程虚拟内存空间中划分一段虚拟内存区域 VMA 出来，并将 VMA 该初始化的属性初始化好，mmap 系统调用就结束了，这里和物理内存还没有发生任何关系
-   私有文件映射：调用 `mmap` 进行内存文件映射的时候可以通过指定参数 `flags` 为 `MAP_PRIVATE`，然后将参数 `fd` 指定为要映射文件的文件描述符来实现对文件的私有映射
-   共享文件映射：通过将 `mmap` 系统调用中的 `flags` 参数指定为 `MAP_SHARED` , 参数 `fd` 指定为要映射文件的文件描述符来实现对文件的共享映射，共享文件映射其实和私有文件映射前面的映射过程是一样的，唯一不同的点在于**私有文件映射仅仅是读共享的，写的时候会发生写时复制（copy on write），并且多进程针对同一映射文件的修改不会回写到磁盘文件上**。而共享文件映射因为是共享的，多个进程中的虚拟内存映射区最终会通过缺页中断的方式映射到文件的 page cache 中，后续多个进程对各自的这段虚拟内存区域的读写都会直接发生在 page cache 上。因为映射文件的 page cache 在内核中只有一份，所以对于共享文件映射来说，多进程读写都是共享的，由于多进程直接读写的是 page cache ，所以多进程对共享映射区的任何修改，最终都会通过内核回写线程 pdflush 刷新到磁盘文件中
-    共享匿名映射：通过将 `mmap` 系统调用中的 `flags` 参数指定为 `MAP_SHARED | MAP_ANONYMOUS`，并将 `fd` 参数指定为 `-1` 来实现共享匿名映射，常用于父子进程之间共享内存通信

例如，基于`mmap`实现共享内存（文件映射）的示例代码如下：

```CPP
int start_mmap(){
    /*
    #define SHM_SIZE 1024
    int fd = shm_open(SHM_NAME, O_CREAT | O_RDWR, 0666);  // POSIX共享内存文件
    ftruncate(fd, SHM_SIZE);
    */
    int fd = open(name, O_RDWR | O_CREAT, 0666);	

	if (fd == -1) {
		exit(-1);
	}

	//set total_size
	TotalSizeInit();

	ret = ftruncate(fd, total_size);
	if (ret == -1) {
		exit(-1);
	} 

	mem_base = (char *)mmap(NULL, total_size,
				PROT_READ | PROT_WRITE,
				MAP_SHARED, fd, 0); 
	if (mem_base == MAP_FAILED) {
		exit(-1);
	}

	if (mlock_open_flag == OPEN_MLOCK) {	
		ret = mlock(mem_base, total_size);
		if (ret == -1) {
			exit(-1);
		} 
	}
    //....
}
```

####    mmap的原理（内核视角）
mmap 系统调用的本质是首先要在进程虚拟内存空间里的文件映射与匿名映射区中划分出一段虚拟内存区域 VMA 出来 ，这段 VMA 区域的大小用 `vm_start`/`vm_end` 来表示，它们由 `mmap` 系统调用参数 `addr`/`length` 决定；进一步说，`mmap` 内存文件映射的本质其实就是将虚拟映射区 vma 的相关操作 `vma->vm_ops` 映射成文件的相关操作 `ext4_file_vm_ops`（以ext4文件系统为例）

```CPP
struct vm_area_struct {
    unsigned long vm_start;     /* Our start address within vm_mm. */
    unsigned long vm_end;       /* The first byte after our end address */

    ......

    //with fd
    struct file * vm_file;      /* File we map to (can be NULL). */
    unsigned long vm_pgoff;     /* Offset (within vm_file) in PAGE_SIZE */
}
```

随后内核会对该段 VMA 进行相关的映射，如果是文件映射的话，内核会将要映射的文件，以及要映射的文件区域在文件中的 `offset`，与 VMA 结构中的 `vm_file`/`vm_pgoff` 关联映射起来，它们由 `mmap` 系统调用参数 `fd`/`offset` 决定。

由 `mmap` 在文件映射与匿名映射区中映射出来的这一段虚拟内存区域同进程虚拟内存空间中的其他虚拟内存区域一样，也都是有权限（读/写/可执行等）控制的

####    mmap与VMA的访问权限控制
![mmap-access-control]()

以上图的进程虚拟内存空间为例：

-  代码段：它是与磁盘上 ELF 格式可执行文件中的 `.text` section（磁盘文件中各个区域的单元组织结构）进行映射的，存放的是程序执行的机器码，所以在可执行文件与进程虚拟内存空间进行文件映射的时候，需要指定代码段这个虚拟内存区域的权限为可读（`VM_READ`）与可执行的（`VM_EXEC`）权限
-   数据段：也是通过文件映射进来的，内核会将磁盘上 ELF 格式可执行文件中的 `.data` section 与数据段映射起来，在映射的时候需要指定数据段这个虚拟内存区域的权限为可读（`VM_READ`）与可写（`VM_WRITE`）权限
-   BSS段、堆、栈等：这些虚拟内存区域并不是从磁盘二进制可执行文件中加载的，它们是通过匿名映射的方式映射到进程虚拟内存空间的，其中BSS 段中存放的是程序未初始化的全局变量，这段虚拟内存区域的权限是可读（`VM_READ`）与可写（`VM_WRITE`）权限；堆是用来描述进程在运行期间动态申请的虚拟内存区域的，所以通常堆也会具有可读（`VM_READ`）与可写（`VM_WRITE`）权限；栈是用来保存进程运行时的命令行参数、环境变量以及函数调用过程中产生的栈帧的，栈一般拥有可读（`VM_READ`）与可写（`VM_WRITE`）权限
-   文件映射与匿名映射区：由于每调用一次 mmap，无论是匿名映射也好还是文件映射也好，通常都会在文件映射与匿名映射区里产生一个 VMA，相关权限由`mmap`的参数 `prot/flags`控制，最终会映射到虚拟内存区域 VMA 结构中的 `vm_page_prot/vm_flags` 成员，指定进程对这块虚拟内存区域的访问权限和相关标志位
-   此外，进程运行过程中所依赖的动态链接库 `.so` 文件，也是通过文件映射的方式将动态链接库中的代码段，数据段映射进文件映射与匿名映射区中

```CPP
struct vm_area_struct {
    /*
     * Access permissions of this VMA.
     */
    pgprot_t vm_page_prot;
    unsigned long vm_flags; 
}
```

`mmap` 的参数 `prot` 来指定其在进程虚拟内存空间中映射出的这段虚拟内存区域 VMA 的访问权限：

```CPP
#define PROT_READ	0x1		/* 表示该虚拟内存区域背后映射的物理内存是可读 */
#define PROT_WRITE	0x2		/* 表示该虚拟内存区域背后映射的物理内存是可写的 */
#define PROT_EXEC	0x4		/* 表示该虚拟内存区域背后映射的物理内存所存储的内容是可以被执行的，该内存区域内往往存储的是执行程序的机器码，比如进程虚拟内存空间中的代码段，以及动态链接库通过文件映射的方式加载进文件映射与匿名映射区里的代码段，这些 VMA 的权限就是 PROT_EXEC  */
#define PROT_NONE	0x0		/* 表示这段虚拟内存区域是不能被访问的，既不可读写，也不可执行。用于实现防范攻击的 guard page。如果攻击者访问了某个 guard page，就会触发 SIGSEV 段错误。除此之外，指定 PROT_NONE 还可以为进程预先保留这部分虚拟内存区域，虽然不能被访问，但是当后面进程需要的时候，可以通过 mprotect 系统调用修改这部分虚拟内存区域的权限 */
```

####    mmap与VMA的映射方式
虚拟内存映射区域 VMA 的映射方式由 `mmap` 系统调用参数 `flags` 决定，常见如下：

```CPP
#define MAP_FIXED   0x10        /* Interpret addr exactly */
#define MAP_ANONYMOUS   0x20        /* don't use a file */
#define MAP_SHARED  0x01        /* Share changes */
#define MAP_PRIVATE 0x02        /* Changes are private */
```

-   `flags`设定为`MAP_ANONYMOUS`：表示匿名映射（虚拟内存对匿名物理内存页的映射），`fd`与`offset`无意义
-   反之，则表示文件映射（虚拟内存对文件页的映射），需指定`fd`与`offset`
-   根据 `mmap` 创建出的这片虚拟内存区域背后所映射的物理内存能否在多进程之间共享，又分为了两种内存映射方式：
    -   `MAP_SHARED`：表示共享映射，通过 mmap 映射出的这片内存区域在多进程之间是共享的，一个进程修改了共享映射的内存区域，其他进程是可以看到的，用于多进程之间的通信
    -   `MAP_PRIVATE`：表示私有映射，通过 mmap 映射出的这片内存区域是进程私有的，其他进程是看不到的。如果是私有文件映射，那么多进程针对同一映射文件的修改将不会回写到磁盘文件上

由此引申出`mmap`常见的四种应用场景：

-   私有匿名映射，其主要用于进程申请虚拟内存，以及初始化进程虚拟内存空间中的 BSS 段，堆，栈这些虚拟内存区域
-   私有文件映射，特点是**背后映射的文件页在多进程之间是读共享的，多个进程对各自虚拟内存区的修改只能反应到各自对应的文件页上，而且各自的修改在进程之间是互不可见的，最重要的一点是这些修改均不会回写到磁盘文件中**（常用于加载二进制可执行文件的 `.text/.data`等 section 到进程虚拟内存空间中的代码段和数据段中）
-   共享文件映射，多进程之间读写共享（不会发生写时复制），常用于多进程之间共享内存（page cache）通信
-   共享匿名映射，用于父子进程之间共享内存，父子进程之间的通讯。父子进程之间需要依赖 tmpfs 中的匿名文件来实现共享内存

####    mmap文件映射的原理

![mmap-file-mapping](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/memory/mmap/mmap-file-mapping.png)

由上图，调用 `mmap` 进行内存文件映射的时候，内核首先会在进程的虚拟内存空间中创建一个新的虚拟内存区域 VMA 用于映射文件，通过 `vm_area_struct->vm_file` 将映射文件的 `struct file` 结构与虚拟内存映射关联起来，即下面的成员：

```CPP
struct vm_area_struct {
    //...
    struct file * vm_file;      /* File we map to (can be NULL). */
    unsigned long vm_pgoff;     /* Offset (within vm_file) in PAGE_SIZE */
    // ...
}

struct inode {
    //...
    struct address_space	*i_mapping;
}

//page cache 在内核中是使用 struct address_space 结构来描述
//page cache 是和文件相关的，与进程无关
struct address_space {
    // 这里就是 page cache，里边缓存了文件的所有缓存页面
    struct radix_tree_root  page_tree; 
}
```

根据[Linux 内核之旅（二）：VFS（基础篇）](https://pandaychen.github.io/2024/11/20/A-LINUX-KERNEL-TRAVEL-3/)一文的介绍，`vm_file->f_inode` 可以关联到映射文件的 `struct inode`，近而关联到映射文件在磁盘中的磁盘块 `i_block`以及对应的page cache成员`i_mapping`等（内核会将已经访问过的磁盘块缓存在page cache中）

在VFS中，映射文件中的数据是按照磁盘块来存储的，读写文件数据也是按照磁盘块为单位进行的，磁盘块大小为 `4K`，当进程读取磁盘块的内容到内存之后，站在内存管理系统的视角，磁盘块中的数据被 DMA 拷贝到了物理内存页（即文件页）中。一个文件包含多个磁盘块，当它们被读取到内存之后，一个文件也就对应了多个文件页，这些文件页在内存中统一被 page cache 的结构所组织。每一个文件在内核中都会有一个唯一的 page cache 与之对应，用于缓存文件中的数据

##  0x04    mmap应用场景

####    方式1：私有匿名映射
-   场景1：`malloc`申请大块私有（虚拟）内存
-   场景2：在`execve`系统调用中，在当前进程中加载并执行一个新的二进制执行文件

`flags`设置为`MAP_PRIVATE | MAP_ANONYMOUS` 表示私有匿名映射，通常会使用此映射方式来申请虚拟内存，如使用 `malloc` 进行虚拟内存申请时，当申请的内存大于 [`128K`](https://elixir.bootlin.com/glibc/glibc-2.42.9000/source/malloc/malloc.c#L951) 时，`malloc` 就会调用 `mmap` 采用私有匿名映射的方式来申请堆内存。因为它是私有的，所以申请到的内存是进程独占的，多进程之间不能共享。再次强调 `mmap` 私有匿名映射申请到的只是虚拟内存，内核只是在进程虚拟内存空间中划分一段 VMA 出来，并初始化好，`mmap` 系统调用就结束了（当前这里和物理内存还没有任何关系）

1、申请大块内存&&访问

Linux内核中的页表会占用物理内存。由于页表本身也是内核数据结构，需要存储在物理内存中才能被MMU（内存管理单元）访问。内核在创建进程时，只创建了进程的全局页表PGD（渐进式页表分配策略），并不是在进程创建时就分配所有页表，而是在发生缺页中断时按需分配

每个进程都有自己独立的页表，存储在物理内存中，Linux x86_64体系下的四级页表的构造如下：

```TEXT
虚拟地址: [63:48] [47:39] [38:30] [29:21] [20:12] [11:0]
         符号扩展  PML4    PDPT    PD      PT      偏移
```

当进程开始访问这段虚拟内存区域时，发现这段虚拟内存区域背后没有任何物理内存与其关联，体现在内核中就是这段虚拟内存地址在页表中的 PTE  项是空的，亦或者其 PTE 中的 `P` 位为 `0` ，这些都是表示虚拟内存还未与物理内存进行映射，参考下面两张图

![mmap-case1-pte-empty-1]()

![mmap-case1-pte-empty-2]()

这时 MMU 就会触发缺页（缺少物理内存页）异常（page fault），随后进程就会切换到内核态，在内核缺页中断处理程序中，为这段虚拟内存区域分配对应大小的物理内存页，随后将物理内存页中的内容全部初始化为 `0` ，最后在页表中建立虚拟内存与物理内存的映射关系，缺页异常处理结束。当缺页处理程序返回时，CPU 会重新启动引起本次缺页异常的访存指令，这时 MMU 就可以正常翻译出物理内存地址了

![mmap-case1-private-memory-alloc]()

2、`execve`系统调用中的`mmap`行为

`execve` 系统调用（`execve(const char* filename, const char* argv[], const char* envp[])`）的作用是在当前进程中加载并执行一个新的二进制执行文件，参数 `filename` 指定新的可执行文件的文件名，`argv` 用于传递新程序的命令行参数，`envp` 用来传递环境变量

既然是在当前进程中重新执行一个程序，那么当前进程的用户态虚拟内存空间就没有用了，内核需要根据这个可执行文件重新映射进程的虚拟内存空间。大致步骤为内核先删除释放旧的虚拟内存空间，并清空进程页表。然后根据 `filename` 打开可执行文件，并解析文件头，判断可执行文件的格式，不同的文件格式需要不同的函数（`load_binary`函数，实例化为`load_elf_binary/load_aout_binary`等）进行加载。在 `load_binary` 中会解析对应格式的可执行文件，并根据文件内容重新映射进程的虚拟内存空间。比如，虚拟内存空间中的 BSS 段/堆/栈等这些内存区域中的内容不依赖于可执行文件，所以在 `load_binary` 中采用私有匿名映射的方式来创建此类VMA

![mmap-case1-execve]()

####    方式2：私有文件映射
调用 `mmap` 实现内存文件映射，可以通过指定参数 `flags` 为 `MAP_PRIVATE`，然后将参数 `fd` 指定为要映射文件的文件描述符。假设多个进程对 `file-read-write.txt`文件实现私有文件映射，从文件 `offset` 偏移处开始，映射 `length` 长度（假设为`4k`，方便与内存页对齐）的文件内容到各个进程的虚拟内存空间中，调用完 `mmap` 之后，相关内存映射内核数据结构关系如下图所示：

![mmap-case2-file-private-mapping]()

1、磁盘块 && 文件页（page cache）

理解私有文件映射机制需要了解下Linux文件系统的基本知识，以ext4文件系统为例，每一个磁盘上的文件在内核中都会有一个唯一的 `struct inode` 结构，inode 结构和进程是没有关系的，一个文件在内核中只对应一个 inode，inode 结构用于描述文件的元信息（如文件权限、文件中包含多少个磁盘块、每个磁盘块位于磁盘中的什么位置等等）。Linux 是按照磁盘块（相邻的扇区，扇区单位`512`字节）为单位（`4k`）对磁盘中的数据进行管理的

```CPP
//ext4_inode：ext4 文件系统中的 inode 结构
struct ext4_inode {
   // 文件权限
  __le16  i_mode;    /* File mode */
  // 文件包含磁盘块的个数
  __le32  i_blocks_lo;  /* Blocks count */
  // 存放文件包含的磁盘块
  __le32  i_block[EXT4_N_BLOCKS];/* Pointers to blocks */
};
```

因此，调用 `mmap` 进行虚拟内存文件映射的时候，内核首先会在进程的虚拟内存空间中创建一个新的虚拟内存区域 VMA 用于映射文件，通过 `vm_area_struct->vm_file` 将映射文件的 `struct flle` 指针与虚拟内存映射绑定。如此，根据 `vm_file->f_inode` 可以关联到映射文件的 `struct inode`，近而关联到映射文件在磁盘中的磁盘块 `i_block`（进而寻址到文件在磁盘上的存储内容），所以`mmap` 内存文件映射的本质就是建立起虚拟内存区域 VMA 到文件磁盘块之间的映射关系

```CPP
struct vm_area_struct {
    struct file * vm_file;      /* File we map to (can be NULL). */
    unsigned long vm_pgoff;     /* Offset (within vm_file) in PAGE_SIZE */
}
```

进一步说，一个磁盘上的文件（通常）包含多个磁盘块，当它们被读取到内存之后，一个文件也就对应了多个文件（内存）页，也叫做page cache，同时内核会将已经访问过的磁盘块缓存在文件页中。每一个文件在内核中都会有唯一的 page cache 与之对应，用于缓存文件中的数据，page cache 与文件相关，与进程无关，即多进程可以打开同一个文件，每个进程中都有一个 `struct file` 结构来描述这个文件，但是一个文件在内核中只会对应一个 page cache

VFS的`struct inode`结构中，关联page cache的成员主要是`i_mapping`，page cache 在内核中是使用 `struct address_space` 结构来描述的。page cache 在内核中是使用基树 radix_tree 结构来表示的，简单描述下即文件页（page cache）是挂在 radix_tree 的叶子结点上，radix_tree 中的 `root` 节点和 `node` 节点是文件页（叶子节点）的索引节点

TODO

```CPP
struct inode {
    ......
    struct address_space	*i_mapping;
}

struct address_space {
    // 这里就是 page cache，里边缓存了文件的所有缓存页面
    struct radix_tree_root  page_tree;
    ......
}
```

![mmap-case2-all-sight]()

2、多进程对文件映射区的访问（读取）过程

和私有内存映射一样，当多个进程调用 `mmap` 对磁盘上同一个文件进行私有文件映射的时候，内核只是在每个进程的虚拟内存空间中创建出一段虚拟内存区域 VMA 出来，同时将该VMA与文件`fd`映射起来，`mmap` 系统调用就返回了（此时page cache可能都是空的）

当进程 `1` 使用虚拟内存地址开始访问这段VMA时，CPU 会把虚拟内存地址送到 MMU 中进行地址翻译，一般情况下会触发缺页中断（异常）创建相应的页表项、PTE等，进程切换到内核态，在内核缺页中断处理程序中会发现引起缺页的这段 VMA 是私有文件映射的，所以内核会首先通过 `vm_area_struct->vm_pgoff` 在文件 page cache 中查找是否有缓存相应的文件页（映射的磁盘块对应的文件页）；如果文件页不在 page cache 中，内核则会在物理内存中分配一个内存页，然后将新分配的内存页加入到 page cache 中，并增加页引用计数。随后会通过 `address_space_operations` 中定义的 `readpage` 激活块设备驱动从磁盘中读取映射的文件内容，然后将读取到的内容填充新分配的内存页

```CPP
struct vm_area_struct {
    unsigned long vm_pgoff;     /* Offset (within vm_file) in PAGE_SIZE */
}

static inline struct page *find_get_page(struct address_space *mapping,
     pgoff_t offset)
{
   return pagecache_get_page(mapping, offset, 0, 0);
}

static const struct address_space_operations ext4_aops = {
    .readpage       = ext4_readpage         //ext4文件系统
}
```

此时文件中映射的内容已经加载进 page cache（物理内存） 了，**在缺页中断处理程序的最后一步，内核会为映射的这段虚拟内存在页表中创建 PTE，然后将虚拟内存与 page cache 中的文件页通过 PTE 关联起来，缺页处理就结束了**，注意到这里私有文件映射下 PTE 中文件页的权限是只读的

![mmap-case2-pte-readonly]()

当进程`1`完成了文件映射之后，进程 `1` 中的页表已经建立起了虚拟内存与文件页的映射关系，因此当进程 `1` 再次访问这段虚拟内存的时候，其实就等于直接访问文件的 page cache。**整个过程是在用户态进行的，不需要切态**，这里就解释了为何mmap的速度非常快

![mmap-case2-process1-done]()

继续，当另一个进程`2`使用相同的参数加载私有文件映射时，同样会产生缺页中断填充响应的次级页表项，不同的是此时被映射的文件内容已经加载到 page cache 中了，进程 `2` 只需要创建 PTE ,并将 page cache 中的文件页与进程 `2` 映射的这段虚拟内存通过 PTE 关联起来即可（同样，进程 `2` 的 PTE 也是readonly）。此刻进程 `1` 和进程 `2` 都可以根据各自虚拟内存空间中映射的这段虚拟内存对文件的 page cache 进行读取了，整个过程都发生在用户态，不需要切态，因为虚拟内存现在已经直接映射到 page cache 了（如果多进程场景下只是对文件映射部分进行读取操作，文件页其实在多进程之间是共享的，整个内核中只有一份）

3、私有映射场景下对共享文件区的写入？

虽然通过 `mmap` 映射的时候指定的这段VMA是可写的，但是由于采用的是私有文件映射的方式，各个进程页表中对应 PTE 却是只读的，**当任意进程对这段VMA进行写入的时候，MMU 会发现 PTE 是只读的，所以会产生一个写保护类型的缺页中断，此时执行写入进程（如进程 `1`）此时又会陷入到内核态，在写保护缺页处理中，内核会重新申请一个内存页，然后将 page cache 中的内容拷贝到这个新的内存页中，进程 `1` 页表中对应的 PTE 会重新关联到这个新的内存页上，此时 PTE 的权限变为可写**。此后进程 `1` 对这段VMA进行读写的时候就不会再发生缺页了，读写操作都会发生在这个新申请的内存页上，并且进程 `1` **对这个内存页的任何修改均不会回写到磁盘文件上**，这也体现了私有文件映射的特点，进程对映射文件的修改，其他进程是看不到的，并且修改不会同步回磁盘文件中

当进程`2`也执行相同的操作后，此时内存布局如下：

![mmap-case2-multiple-write]()

4、应用场景

如上，进程 `1` 和进程 `2` 对各自虚拟内存区的修改只能反应到各自对应的物理内存页上，而且各自的修改在进程之间是互不可见的，这些修改均不会回写到原始的磁盘文件中。可以利用 `mmap` 私有文件映射的这个特性，来加载二进制可执行文件的 `.text/.data` 等 section 到进程虚拟内存空间中的代码段和数据段中（对于二进制加载的场景，多进程之间对数据段的修改相互之间是不可见的，而且对数据段的修改不能回写到磁盘上的二进制文件中）

```CPP
//加载 a.out 格式可执行文件
static int load_aout_binary(struct linux_binprm * bprm)
{
    ......
    // 将 .text 采用私有文件映射的方式映射到进程虚拟内存空间的代码段
    error = vm_mmap(bprm->file, N_TXTADDR(ex), ex.a_text,
        PROT_READ | PROT_EXEC,
        MAP_FIXED | MAP_PRIVATE | MAP_DENYWRITE | MAP_EXECUTABLE,
        fd_offset);

    // 将 .data 采用私有文件映射的方式映射到进程虚拟内存空间的数据段
    error = vm_mmap(bprm->file, N_DATADDR(ex), ex.a_data,
            PROT_READ | PROT_WRITE | PROT_EXEC,
            MAP_FIXED | MAP_PRIVATE | MAP_DENYWRITE | MAP_EXECUTABLE,
            fd_offset + ex.a_text);

    ......
}
```

![mmap-case2-elf-load]()

####    方式3：共享文件映射（多进程通信）
共享文件映射的参数为`MAP_SHARED`（`vm_area_struct->vm_flags`），其中参数 `fd` 指定为要映射文件的文件描述符。与私有文件映射的过程类似，不同之处在于私有文件映射是读共享的，写的时候会发生写时复制（copy on write），并且多进程针对同一映射文件的修改不会回写到磁盘文件上。而共享文件映射中多个进程中的虚拟内存映射区最终会通过缺页中断的方式映射到文件的 page cache 中，后续多个进程对各自的这段虚拟内存区域的读写都会直接发生在 page cache 上，**因为映射文件的 page cache 在内核中只有一份，所以对于共享文件映射来说，多进程读写都是共享的，由于多进程直接读写的是 page cache ，所以多进程对共享映射区的任何修改，最终都会通过内核回写线程 pdflush 刷新到磁盘文件中**

![mmap-case3-multiple-rw]()

继续，当某个进程触发虚拟内存地址访问时，共享文件映射场景下，PTE 被创建出来的时候就是可写的，所以后续该进程在对这段虚拟内存区域写入的时候不会触发缺页中断，而是直接写入 page cache 中，整个过程没有切态，没有数据拷贝

![mmap-case3-multiple-rw-2]()

所以，共享内存映射的本质是现在进程 `1` 和进程 `2` 各自虚拟内存空间中的这段虚拟内存区域 VMA，已经共同映射到了文件的 page cache 中，由于文件的 page cache 在内核中只有一份，它是和进程无关的，page cache 中的内容发生的任何变化，进程 `1` 和进程 `2` 都是可以看到的。最后，多进程对各自虚拟内存映射区 VMA 的写入操作，内核会根据脏页回写策略将修改内容回写到磁盘文件中，策略如下：

-   `dirty_writeback_centisecs`：内核参数的默认值为 `500`（单位为 `0.01s`），也就是说内核默认会每隔 `5s` 唤醒一次 flusher 线程来执行相关脏页的回写
-   `drity_background_ratio`：当脏页数量在系统的可用内存 available 中占用的比例达到 `drity_background_ratio` 的配置值时，内核就会唤醒 flusher 线程异步回写脏页。默认值为`10`，表示如果 page cache 中的脏页数量达到系统可用内存的 `10%` 的话，就主动唤醒 flusher 线程去回写脏页到磁盘
-   `dirty_background_bytes`：如果 page cache 中脏页占用的内存用量绝对值达到指定的 `dirty_background_bytes`。内核就会唤醒 flusher 线程异步回写脏页。默认为`0`
-   `dirty_ratio` ： `dirty_background_*` 相关的内核配置参数均是内核通过唤醒 flusher 线程来异步回写脏页。下面要介绍的 `dirty_*` 配置参数，均是由用户进程同步回写脏页。表示内存中的脏页太多了，用户进程自己都看不下去了，不用等内核 flusher 线程唤醒，用户进程自己主动去回写脏页到磁盘中。当脏页占用系统可用内存的比例达到 `dirty_ratio` 配置的值时，用户进程同步回写脏页。默认值为`20`
-   `dirty_bytes` ：如果 page cache 中脏页占用的内存用量绝对值达到指定的 `dirty_bytes`，用户进程同步回写脏页。默认值为`0`
-   内核为了避免 page cache 中的脏页在内存中长久的停留，所以会给脏页在内存中的驻留时间设置一定的期限（`dirty_expire_centisecs`），超过 `dirty_expire_centisecs` 之后，flusher 线程将会在下次被唤醒的时候将这些脏页回写到磁盘中

####    方式4：共享匿名（文件）映射
共享匿名映射看作成一种特殊的共享文件映射方式，可以用于父子（亲缘）进程通信，`flags`设置为`MAP_SHARED|MAP_ANONYMOUS`，`fd`设置为`-1`，内核通过`tmpfs`文件系统创建出一个匿名文件，匿名文件也有自己的 inode 结构以及 page cache，但是对用户是不可见的，其他过程与共享文件映射完全相同

既然这个基于`tmpfs`的匿名文件对用户不可见，那么就不太可能用于非亲缘关系的进程通信场景了，而父子进程场景下，父进程可以通过`fork`创建子进程，子进程会拷贝父进程的所有资源，当然也包括父进程的虚拟内存空间以及父进程的页表，关联函数[`copy_mm`](https://elixir.bootlin.com/linux/v4.11.6/source/kernel/fork.c#L1174)，参考前文[Linux 内核之旅（一）：进程](https://pandaychen.github.io/2024/10/02/A-LINUX-KERNEL-TRAVEL-1/)

![mmap-case4-anonymous-mapping]()

当 fork 出子进程的时候，这时子进程的虚拟内存空间和父进程的虚拟内存空间完全是一模一样的，在子进程的虚拟内存空间中自然也有一段虚拟映射区 VMA 并且已经关联到匿名文件中了（继承自父进程）

####    大页内存映射

##  0x05  参考
-   [4.6 深入理解 Linux 虚拟内存管理](https://www.xiaolincoding.com/os/3_memory/linux_mem.html)
-   [4.7 深入理解 Linux 物理内存管理](https://www.xiaolincoding.com/os/3_memory/linux_mem2.html#_6-1-%E5%8C%BF%E5%90%8D%E9%A1%B5%E7%9A%84%E5%8F%8D%E5%90%91%E6%98%A0%E5%B0%84)
-   [mmap 源码分析](https://leviathan.vip/2019/01/13/mmap%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/)
-   [linux源码解读（十六）：红黑树在内核的应用虚拟内存管理](https://www.cnblogs.com/theseventhson/p/15820092.html)
-   [图解 Linux 虚拟内存空间管理](https://github.com/liexusong/linux-source-code-analyze/blob/master/process-virtual-memory-manage.md)
-   [认真分析mmap：是什么为什么怎么用](https://www.cnblogs.com/huxiao-tee/p/4660352.html)
-   [从内核世界透视 mmap 内存映射的本质（原理篇）](https://www.cnblogs.com/binlovetech/p/17712761.html)
-   [聊聊Linux内核](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=Mzg2MzU3Mjc3Ng==&action=getalbum&album_id=2559805446807928833&scene=173&from_msgid=2247486879&from_itemidx=1&count=3&nolastread=1#wechat_redirect)
-   [一步一图带你深入理解 Linux 虚拟内存管理](https://mp.weixin.qq.com/s?__biz=Mzg2MzU3Mjc3Ng==&mid=2247486732&idx=1&sn=435d5e834e9751036c96384f6965b328&chksm=ce77cb4bf900425d33d2adfa632a4684cf7a63beece166c1ffedc4fdacb807c9413e8c73f298&token=1931867638&lang=zh_CN#rd)
-   [你写的代码是如何跑起来的](https://mp.weixin.qq.com/s/1bdktqYF7VyAMadRlcRrSg)