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

用户进程在能够读/写一个文件之前必须要先open这个文件。对文件的读/写从概念上说是一种进程与文件系统之间的一种有连接通信，所谓打开文件实质上就是在进程与文件之间建立起链接。在文件系统的处理中，每当一个进程重复打开同一个文件时就建立起一个由`struct file`结构代表的独立的上下文。通常一个`file`结构，即一个读/写文件的上下文，都由一个打开文件号（fd）加以标识

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


##  0x0 参考
-   [open系统调用（一）](https://www.kerneltravel.net/blog/2021/open_syscall_szp1/)
http://blog.chinaunix.net/uid-20522771-id-4419678.html
-   [](https://blog.csdn.net/blue95wind/article/details/7472350)
-   [](https://www.cnblogs.com/jimbo17/p/10491223.html)
-   [linux 内核open文件流程](https://zhuanlan.zhihu.com/p/471175983)