---
layout:     post
title:      eBPF tracing
subtitle:   eBPF与可观测
date:       2026-01-13
author:     pandaychen
catalog:    true
tags:
    - eBPF
---

##  0x00  前言

##  0x01    内核安全策略模块通用框架 LSM

####    LSM 框架介绍
LSM（Linux Security Module）中文翻译为内核安全模块，尽管从名字上是安全模块，但 LSM 其实是一个在内核各个安全模块的基础上提出（抽象出）的轻量级安全访问控制框架。该框架只是提供一个支持安全模块的接口，本身不能增强系统安全性，具体的工作交给各安全模块来做。通俗一点讲 LSM 只是搭建了一个舞台，唱戏的主角还得有具体实现的模块来捧场

LSM 在内核中体现为一组安全相关的函数，是提供实施强制访问控制（MAC）模块的必要组件，可实现策略与内核源代码解耦。安全函数在系统调用的执行路径中会被调用，对用户态进程访问内核资源对象进行强制访问控制，受保护的对象类型包括文件、目录、任务对象、凭据等。从版本 5.4 开始，该框架目前包括整个内核的 224 个挂钩点、一个用于注册要在这些挂钩点调用的函数的 API，以及一个用于保留与受保护内核对象关联的内存以供 LSM 使用的 API

截止到内核 5.7 版本，共有 9 个 LSM 安全模块实现：SELinux、SMACK、AppArmor、TOMOYO、Yama、LoadPin、SafeSetID 和 Lockdown 和 BPF（5.7 版本支持） 。安全模块主要实现如下图所示：

![lsm-history]()

##  0x0    参考
-   [LSM，Linux 内核的安全防护盾](https://mp.weixin.qq.com/s/YRQOmthKgR7mgm7zFVViEA)
-   [LSM BPF 实践](https://www.ebpf.top/post/lsm_bpf_intro/)
-   [利用eBPF LSM实现Blocking写文件操作](https://wujiuye.com/article/35d6c66a23d543888b4f9ed1c31a6f8b)
-   [Searchable Linux Syscall Table](https://filippo.io/linux-syscall-table/)