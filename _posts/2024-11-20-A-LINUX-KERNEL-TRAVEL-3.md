---
layout:     post
title:  Linux 内核之旅（二）：VFS
subtitle:
date:       2024-11-20
author:     pandaychen
header-img:
catalog: true
tags:
    - Linux
---

##  0x00    前言
Linux支持多种文件系统格式（如ext2、ext3、reiserfs、FAT、NTFS、iso9660等），不同的磁盘分区或其它存储设备都有不同的文件系统格式，然而这些文件系统都可以`mount`到某个目录下，使开发者看到一个统一的目录树，各种文件系统上的目录和文件，读写操作用起来也都是一样的。Linux内核在各种不同的文件系统格式之上做了一个抽象层，使得文件、目录、读写访问等概念成为抽象层的概念，因此各种文件系统看起来用起来都一样，这个抽象层称为虚拟文件系统（VFS，Virtual Filesystem）

####	基础结构
![vfs-1](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/file-system/task_struct_fs.png)

-	`inode`：[定义](https://elixir.bootlin.com/linux/v3.4/source/include/linux/fs.h#L761)，与磁盘上真实文件的一对一映射
-	`file`：[定义](https://elixir.bootlin.com/linux/v3.4/source/include/linux/fs.h#L976)，一个 `file` 结构体代表一个物理文件的上下文，不同的进程，甚至同一个进程可以多次打开一个文件，因此具有多个 `file struct`
-	`dentry`：[定义](https://elixir.bootlin.com/linux/v3.4/source/include/linux/dcache.h#L88)，多个 `dentry` 可以指向同一个 `inode`

#### 内核数据结构


##  0x0	 VFS 关联task_struct

##	0x0	files_struct
![files_struct](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/vfs/fs.vfs.png)


##  0x0  参考
-   [VFS](https://akaedu.github.io/book/ch29s03.html)
-   [vfs](https://hushi55.github.io/2015/10/19/linux-kernel-vfs)
-	[File Descriptor Table](https://chenshuo.com/notes/kernel/data-structures/)
-	[Linux的虚拟文件系统VFS](https://sq.sf.163.com/blog/article/218133060619399168)
-   [linux vfs轮廓](https://qiankunli.github.io/2018/05/19/linux_file_system.html)
-   [从文件 I/O 看 Linux 的虚拟文件系统](https://jarsonfang.github.io/Kernel/%E5%86%85%E6%A0%B8%E6%96%87%E4%BB%B6%E7%B3%BB%E7%BB%9F/linux-vfs/)