---
layout:     post
title:  Linux 内核之旅（二十）：内核视角下的共享内存（TODO）
subtitle:   
date:       2025-09-02
author:     pandaychen
header-img:
catalog: true
tags:
    - Linux
    - Kernel
---

##  0x00    前言
共享内存主要用于进程间通信，常见Shared Memory机制：

-   System V shared memory(`shmget/shmat/shmdt`)：旧
-   POSIX shared memory（`shm_open/shm_unlink`）：新

此外，内存映射`mmap`机制也可以用于跨进程间通信，参考前文：[]()

```TEXT
Shared file mappings：Sharing between unrelated processes, backed by file in filesystem
```

####    tmpfs

```BASH
[root@X-X-01 corefile]# df -Th
Filesystem      Type      Size  Used Avail Use% Mounted on
devtmpfs        devtmpfs   16G  4.0K   16G   1% /dev
tmpfs           tmpfs      16G   39M   16G   1% /dev/shm
tmpfs           tmpfs      16G  266M   16G   2% /run
tmpfs           tmpfs      16G     0   16G   0% /sys/fs/cgroup
/dev/vda1       ext4       99G   18G   77G  19% /
tmpfs           tmpfs     3.2G     0  3.2G   0% /run/user/0
/dev/vdb        ext4       98G   47G   47G  50% /data
X.X.X.X:/ nfs4      2.5T  1.9T  642G  75% /data/cfs/xxx
tmpfs           tmpfs     3.2G     0  3.2G   0% /run/user/30000
```

####    共享内存与tmpfs的关系
POSIX共享内存是基于tmpfs来实现的，System V shared memory在内核也是基于tmpfs实现的。tmpfs主要有两个作用：

1.  用于SYSTEM V共享内存，还有匿名内存映射；这部分由内核管理，用户不可见（理解为特殊的文件系统）
2.  用于POSIX共享内存，由用户负责mount，而且一般mount到`/dev/shm`；依赖于`CONFIG_TMPFS`


##  0x01    

##  0x0  参考
-   [深入理解Linux内核共享内存机制- shmem&tmpfs](https://aijishu.com/a/1060000000451498)
-   [浅析Linux的共享内存与tmpfs文件系统](http://hustcat.github.io/shared-memory-tmpfs/)