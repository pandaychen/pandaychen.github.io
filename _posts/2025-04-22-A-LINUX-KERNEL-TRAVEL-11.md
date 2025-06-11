---
layout:     post
title:  Linux 内核之旅（十）：追踪open/write系统调用
subtitle:   VFS
date:       2025-04-02
author:     pandaychen
header-img:
catalog: true
tags:
    - Linux
    - Kernel
---

##  0x00    前言
本文代码基于 [v4.11.6](https://elixir.bootlin.com/linux/v4.11.6/source/include) 版本

用户进程在能够读/写一个文件之前必须要先open这个文件。对文件的读/写从概念上说是一种进程与文件系统之间的一种有连接通信，所谓打开文件实质上就是在进程与文件之间建立起链接。在文件系统的处理中，每当一个进程重复打开同一个文件时就建立起一个由`struct file`结构代表的独立的上下文。通常一个`file`结构，即一个读/写文件的上下文，都由一个打开文件号（fd）加以标识。从VFS的层面来看，open操作的实质就是根据参数指定的路径去获取一个该文件系统（比如`ext4`）的inode（硬盘上），然后触发VFS一系列机制（如生成dentry、加载inode到内存inode以及将dentry指向内存inode等等），然后去填充VFS层的`struct file`结构体，这样就可以让上层使用了

用户态程序调用`open`函数时，会产生一个中断号为`5`的中断请求，其值以该宏`__NR__open`进行标示，而后该进程上下文将会被切换到内核空间，待内核相关操作完成后，就会从内核返回至用户态，此时还需要一次进程上下文切换，本文就以内核视角追踪下`open`（`write`）的内核调用过程

##  0x01   ftrace open系统调用
内核调用链如下（简化）

```BASH
 1) open1-3789488  |               |    __x64_sys_openat() {
 1) open1-3789488  |               |      do_sys_openat2() {
 1) open1-3789488  |               |        do_filp_open() {
 1) open1-3789488  |               |          path_openat() {
 1) open1-3789488  |               |              lookup_open.isra.0() {
 1) open1-3789488  |               |            do_open() {
 1) open1-3789488  |               |              may_open() {
 1) open1-3789488  |               |              vfs_open() {
 1) open1-3789488  |               |                do_dentry_open() {
 1) open1-3789488  |               |                  security_file_open() {
 1) open1-3789488  |               |                    selinux_file_open() {
 1) open1-3789488  |               |                    bpf_lsm_file_open() {
 1) open1-3789488  |               |                  ext4_file_open() {
 1) open1-3789488  |               |                    dquot_file_open() {
 1) open1-3789488  |   0.172 us    |                      generic_file_open();
```

完整的函数调用链[可见]()

##  0x02    再看文件系统缓存

####    dentry cache
一个`struct dentry`结构代表文件系统中的一个目录或文件，VFS dentry结构的意义，即需要建立文件名filename到inode的映射关系，目的为了提高系统调用在操作、查找目录/文件操作场景下的效率，且dentry是仅存在于内存的数据结构，所以内核使用了`dentry_hashtable`（dentry树）来管理整个系统的目录树结构。在Linux可通过下面方式查看dentry cache的指标：

```BASH
[root@VM-X-X-tencentos ~]# cat /proc/sys/fs/dentry-state
1212279 1174785 45      0       598756  0
```

前文已经描述过[`dentry_hashtable`]()数据结构的意义，即为了提高目录，内核使用`dentry_hashtable`对dentry进行管理，在`open`等系统调用进行路径查找时，用于快速定位某个目录dentry下面的子dentry（哈希表的搜索效率显然更好）


1、`dentry`及`dentry_hashtable`的结构（搜索&回收）

对`dentry`而言，对于加速搜索关联了两个关键数据结构`d_hash`及`d_lru`，dentry回收关联的重要成员是`d_lockref`，`d_lockref`内嵌一个自旋锁（spinlock_t），用于保护 dentry 结构的并发修改
```CPP
struct dentry {
    /* Ref lookup also touches following */
	struct lockref d_lockref;	/* per-dentry lock and refcount */

    struct hlist_bl_node d_hash;       //hashtable的节点
    strcut list_head d_lru;     //LRU链上的节点
}
```

![dcache]()

2、`dentry_hashtable`的创建过程

在文件系统初始化时，调用`vfs_caches_init->dcache_init`为dcache进行初始化，先创建一个dentry的slab，用于后续dentry对象的分配，同时还初始化了`dentry_hashtable`这个用于管理dentry的全局hashtable

```CPP
static struct hlist_head *dentry_hashtable __read_mostly;

static void __init dcache_init(void)
{
    /*
     * A constructor could be added for stable state like the lists,
     * but it is probably not worth it because of the cache nature
     * of the dcache.
     */
    dentry_cache = KMEM_CACHE_USERCOPY(dentry,
        SLAB_RECLAIM_ACCOUNT|SLAB_PANIC|SLAB_MEM_SPREAD|SLAB_ACCOUNT,
        d_iname);

    /* Hash may have been set up in dcache_init_early */
    if (!hashdist)
        return;

    //dentry_hashtable 初始化
    dentry_hashtable =
        alloc_large_system_hash("Dentry cache",
                    sizeof(struct hlist_bl_head),
                    dhash_entries,
                    13,
                    HASH_ZERO,
                    &d_hash_shift,
                    NULL,
                    0,
                    0);
    d_hash_shift = 32 - d_hash_shift;
}
```

##  0x03 open流程
一个进程需要读/写一个文件，必须先通过filename建立和文件inode之间的通道，方式是通过`open()`函数，该函数的参数是文件所在的路径名`pathname`，如何根据`pathname`找到对应的inode？这就要依靠dentry结构了

```CPP
int open (const char *pathname, int flags, mode_t mode);
```

##  0x04  write流程
[`vfs_write`](https://elixir.bootlin.com/linux/v4.11.6/source/fs/read_write.c#L542)实现：

```cpp
ssize_t vfs_write(struct file *file, const char __user *buf, size_t count, loff_t *pos)
{
	ssize_t ret;

	if (!(file->f_mode & FMODE_WRITE))
		return -EBADF;
	if (!(file->f_mode & FMODE_CAN_WRITE))
		return -EINVAL;
	if (unlikely(!access_ok(VERIFY_READ, buf, count)))
		return -EFAULT;

	ret = rw_verify_area(WRITE, file, pos, count);
	if (!ret) {
		if (count > MAX_RW_COUNT)
			count =  MAX_RW_COUNT;
		file_start_write(file);
		ret = __vfs_write(file, buf, count, pos);
		if (ret > 0) {
			fsnotify_modify(file);
			add_wchar(current, ret);
		}
		inc_syscw(current);
		file_end_write(file);
	}

	return ret;
}
```

##  0x05 参考
-   [open系统调用（一）](https://www.kerneltravel.net/blog/2021/open_syscall_szp1/)
-   [走马观花： Linux 系统调用 open 七日游](http://blog.chinaunix.net/uid-20522771-id-4419678.html)
-   [Open()函数的内核追踪](https://blog.csdn.net/blue95wind/article/details/7472350)
-   [Linux Kernel文件系统写I/O流程代码分析（二）](https://www.cnblogs.com/jimbo17/p/10491223.html)
-   [linux 内核open文件流程](https://zhuanlan.zhihu.com/p/471175983)
-   [文件系统缓存](https://www.laumy.tech/776.html)
-   [Linux的VFS实现 - 番外[一] - dcache](https://zhuanlan.zhihu.com/p/261669249)
-   [Linux Kernel文件系统写I/O流程代码分析（一）](https://www.cnblogs.com/jimbo17/p/10436222.html)
-   [vfs dentry cache 模块实现分析](https://zhuanlan.zhihu.com/p/457005511)