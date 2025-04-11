---
layout:     post
title:  Linux HIDS 开发场景收集
subtitle:
date:       2024-12-05
author:     pandaychen
header-img:
catalog: true
tags:
    - Linux
    - HIDS
---

##  0x00    场景收集

-   如何根据`task_struct`结构，获取到事件对应运行二进制的绝对路径？
-   如何获取进程打开的文件fd列表？
-   如何根据进程打开的文件fd列表，并且判断其是否为socket网络句柄？
-   如何根据`task_struct`结构，获取该进程对应的进程链信息（类似`pstree`）？
-   如何获取某个进程（`task_struct`）对应的socket五元组信息（如果打开了网络连接）？
-   如何获取某个进程对应的tty信息
-   在`execve`/`execveat`系统调用中，如何获取入参`args`，已经指定的`envp`的值（如`SSH_CONNECTION`等）
-   如何获取`nodename`（区分node与容器场景）
-   如何获取dns请求（基于UDP/TCP发送）：包含dns请求类型`op_type`、dns请求域名等
-   针对`dup/pipe`类型的GLIBC函数，如何找到其关联fd的更多信息，比如类型，路径等
-   基于某些内核hook（如`k[ret]probe/vfs_write`）入参的`struct file *`类型的参数，如何获取其对应的绝对路径？