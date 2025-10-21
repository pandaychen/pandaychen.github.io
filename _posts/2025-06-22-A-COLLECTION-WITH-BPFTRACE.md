---
layout:     post
title:      bpftrace 常用code收集
subtitle:   
date:       2025-06-22
author:     pandaychen
header-img:
catalog: true
tags:
    - eBPF
    - bpftrace
---

##  0x00    前言
本文的bpftrace工具版本：`bpftrace v0.19.1`

####    bpftrace语法
bpftrace语法（类似awk），`{`前的部分相当于awk中的condition，`{}`中的部分相当于awk的action，不过bpftrace执行actions的条件是触发probe名称指定的事件

probe的格式为`namespace:function_name`，如`tracepoint:timer:tick_stop`、`kprobe:do_sys_open`，好处是便于查找函数及定位不同模块中的同名函数。此外，还有两个特殊的probe即`BEGIN`与`END`，这与awk类似，它们分别在bpftrace程序执行开始、结束时，无条件的执行一些操作，比如完成一些初始化、清理工作等

bpftrace亦可使用ebpf支持的probe：`hardware/iter/kfunc/kprobe/software/tracepoint/uprobe`，如本机内核`6.6.30-5.tl4.x86_64`

```BASH
[root@VM-X-X-tencentos ~]# bpftrace -l  | awk -F ":" '{print $1}' | sort | uniq -c | sort  -nr -k 1
  54112 kprobe
  47345 kfunc
   2364 tracepoint
   1676 rawtracepoint
     16 iter
     14 software
     12 hardware
```

####    运行方式
-   `bpftrace -e 'cmds`：执行单行指令
-   `bpftrace filename`：执行脚本文件

```BASH
利用 shell 换行：
[root@localhost ~]# bpftrace -e '
BEGIN{
printf("Hello, World!\n");
}

END{
printf("Bye, World!\n");
}'
#在脚本尾部的单引号之前，按照需要换行即可
```

##  0x01    基础示例

1、helloworld

```BASH
bpftrace -e 'BEGIN {printf("hello world\n");}'
```

2、查询hook点

```BASH
[root@VM-X-X-tencentos ~]# bpftrace -l | grep vfs_read
kfunc:vmlinux:vfs_read
kfunc:vmlinux:vfs_readlink
kfunc:vmlinux:vfs_readv
kprobe:vfs_read
kprobe:vfs_readlink
kprobe:vfs_readv
```

3、打印`pid`、`comm`

```BASH
bpftrace -e 'kprobe:vfs_read {printf("pid = %d, comm = %s\n",pid,comm);}'
```

##  0x02    常用实例

####    静态探针：tracepoint

1、在文件打开的时候打印进程名和文件名

```BASH
[root@VM-119-175-tencentos ~]# bpftrace -e 'tracepoint:syscalls:sys_enter_openat { printf("%s %s\n", comm, str(args.filename)); }'
sshlogd /var/log/sshlog/sessions/ssh_2960655.log
rm /usr/lib64/gconv/gconv-modules.cache
rm /usr/lib/locale/en_US.utf8/LC_MEASUREMENT
rm /usr/lib/locale/en_US.utf8/LC_TELEPHONE
rm /usr/lib/locale/en_US.utf8/LC_ADDRESS
sh /dev/null
rm /usr/lib/locale/en_US.utf8/LC_NAME   
```

-   `comm`是内建变量，代表当前进程的名字。其它类似的变量还有`pid`和`tid`等，分别表示进程标识和线程标识
-   `args`是一个包含所有tracepoint参数的结构。这个结构是由bpftrace根据tracepoint信息自动生成的。这个结构的成员可以通过命令`bpftrace -vl tracepoint:syscalls:sys_enter_openat`查询
-   `args.filename`用来获取`args`的成员变量`filename`的值
-   `str()`：将字符串指针转换成字符串

2、捕获进程调用参数

```BASH
bpftrace -e 'tracepoint:syscalls:sys_enter_execve { printf("%s ", comm); join(args->argv); }'

sh rm -f /tmp/test/697425390.tmp
sh rm -f /tmp/test/694688366.tmp1
sh rm -f /tmp/test/694688367.tmp1
bash readlink /proc/3545309/exe
sh rm -f /tmp/test/694688368.tmp1
bash basename /usr/bin/bash
sh rm -f /tmp/test/694688369.tmp1
```

3、`sys_enter_finit_module`

```BASH
#!/usr/bin/env bpftrace

BEGIN {
    printf("Tracing init_module and finit_module syscalls... Hit Ctrl+C to stop.\n");
}

tracepoint:syscalls:sys_enter_finit_module {
    printf("Syscall executed: %s (PID: %d, UID: %d, Command: %s),umod: %d\n", probe, pid, uid, comm, args->fd);
}
```

4、块级I/O跟踪

```BASH
#运行内核版本：6.6.30-5.tl4.x86_64
bpftrace -e 'tracepoint:block:block_rq_issue { @ = hist(args.bytes); }'
```

输出：
```
@: 
[16, 32)               6 |@                                                   |
[32, 64)               0 |                                                    |
[64, 128)              0 |                                                    |
[128, 256)             0 |                                                    |
[256, 512)             0 |                                                    |
[512, 1K)              0 |                                                    |
[1K, 2K)               0 |                                                    |
[2K, 4K)               0 |                                                    |
[4K, 8K)             233 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@|
[8K, 16K)             22 |@@@@                                                |
[16K, 32K)            25 |@@@@@                                               |
[32K, 64K)            13 |@@                                                  |
[64K, 128K)            4 |                                                    |
[128K, 256K)           4 |                                                    |
```


####    动态探针：kprobe
1、定时调用 `ps` 查看当前进程

```BASH
[root@VM-X-X-centos ~]#  bpftrace --unsafe -e 'kprobe:do_nanosleep { system("ps -p %d\n", pid); }'
Attaching 1 probe...
    PID TTY          TIME CMD
  27511 ?        08:49:56 xx
    PID TTY          TIME CMD
  27511 ?        08:49:56 xx
```

####    uprobe


####    USDT


##  0x03    参考
-   [bpftrace一行教程](https://eunomia.dev/zh/tutorials/bpftrace-tutorial/)
-   [基于ebpf的性能工具-bpftrace脚本语法](https://cloud.tencent.com/developer/article/2322518)