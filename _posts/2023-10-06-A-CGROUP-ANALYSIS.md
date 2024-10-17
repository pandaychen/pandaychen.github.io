---
layout:     post
title:      Linux Namespace && Cgroup
subtitle:   Linux系统资源隔离与管理机制介绍
date:       2023-10-06
author:     pandaychen
catalog:    true
tags:
    - Cgroup
    - Namespace
---

##  0x00    前言
Linux Namespace和Cgroup都是Linux内核的功能，用于隔离和管理系统资源，但它们的目标和方法有所不同

```TEXT
cgroup = limits how much you can use;
namespaces = limits what you can see (and therefore use)
```

-   Linux Namespace主要用于隔离系统资源。这意味着在一个Namespace中的进程可能看不到另一个Namespace中的资源。例如，一个进程可能有自己的网络堆栈，与其他进程的网络堆栈完全隔离。Namespace是实现容器技术的基础，如Docker
-   Cgroup（Control Group）则主要用于管理和限制系统资源的使用。它可以限制一个进程组使用的CPU、内存等资源的数量。例如，你可以设置一个Cgroup，使得其中的进程最多只能使用50%的CPU时间

总结来说，Namespace是隔离资源，使得进程不能看到其他Namespace的资源。而Cgroup是限制资源的使用，即使进程可以看到资源，也可能因为Cgroup的限制而无法使用

##  0x01    Namespace


##  0x02  Cgroup


##  0x03    CGROUP限制实现

##  0x04    一些细节

1、查看pid进程对应的cgroup信息

子进程也会集成父进程的cgroup限制（分组）

```BASH
[root@VM-130-44-centos ~]# cat /proc/12965/cgroup 
12:pids:/collector-XXX-f75df986656263e86157c4df672b81c4
11:perf_event:/collector-XXX-f75df986656263e86157c4df672b81c4
10:cpu,cpuacct:/collector-XXX-f75df986656263e86157c4df672b81c4
9:freezer:/collector-XXX-f75df986656263e86157c4df672b81c4
8:blkio:/collector-XXX-f75df986656263e86157c4df672b81c4
7:cpuset:/collector-XXX-f75df986656263e86157c4df672b81c4
6:misc:/
5:memory:/collector-bkmonitorbeat-f75df986656263e86157c4df672b81c4
4:net_cls,net_prio:/collector-XXX-f75df986656263e86157c4df672b81c4
3:hugetlb:/collector-XXX-f75df986656263e86157c4df672b81c4
2:devices:/collector-XXX-f75df986656263e86157c4df672b81c4
1:name=systemd:/collector-XXX-f75df986656263e86157c4df672b81c4
```

2、查看对应cgroup的CPU限制

下面数据说明该进程最多可以使用`10%`的CPU资源
```BASH
[root@VM-130-44-centos ~]# cat /sys/fs/cgroup/cpu/XXX-bkmonitorbeat-f75df986656263e86157c4df672b81c4/cpu.cfs_period_us
100000
[root@VM-130-44-centos ~]# cat /sys/fs/cgroup/cpu/XXX-bkmonitorbeat-f75df986656263e86157c4df672b81c4/cpu.cfs_quota_us 
1000000
```

3、查看对应cgroup的memory限制

```BASH
cat /sys/fs/cgroup/memory/system.slice/XXX-bkmonitorbeat-f75df986656263e86157c4df672b81c4/memory.limit_in_bytes
```

4、查看默认cgroup挂载分组

```BASH
[root@VM-130-44-centos ~]# mount -t cgroup
cgroup on /sys/fs/cgroup/systemd type cgroup (rw,nosuid,nodev,noexec,relatime,xattr,release_agent=/usr/lib/systemd/systemd-cgroups-agent,name=systemd)
cgroup on /sys/fs/cgroup/devices type cgroup (rw,nosuid,nodev,noexec,relatime,devices)
cgroup on /sys/fs/cgroup/hugetlb type cgroup (rw,nosuid,nodev,noexec,relatime,hugetlb)
cgroup on /sys/fs/cgroup/net_cls,net_prio type cgroup (rw,nosuid,nodev,noexec,relatime,net_cls,net_prio)
cgroup on /sys/fs/cgroup/memory type cgroup (rw,nosuid,nodev,noexec,relatime,memory)
cgroup on /sys/fs/cgroup/misc type cgroup (rw,nosuid,nodev,noexec,relatime,misc)
cgroup on /sys/fs/cgroup/cpuset type cgroup (rw,nosuid,nodev,noexec,relatime,cpuset)
cgroup on /sys/fs/cgroup/blkio type cgroup (rw,nosuid,nodev,noexec,relatime,blkio)
cgroup on /sys/fs/cgroup/freezer type cgroup (rw,nosuid,nodev,noexec,relatime,freezer)
cgroup on /sys/fs/cgroup/cpu,cpuacct type cgroup (rw,nosuid,nodev,noexec,relatime,cpu,cpuacct)
cgroup on /sys/fs/cgroup/perf_event type cgroup (rw,nosuid,nodev,noexec,relatime,perf_event)
cgroup on /sys/fs/cgroup/pids type cgroup (rw,nosuid,nodev,noexec,relatime,pids)
```

##  0x05    参考
-   [Linux Namespace和Cgroup](https://segmentfault.com/a/1190000009732550)
-   [如何在 Go 中使用 CGroup 实现进程内存控制](https://cloud.tencent.com/developer/article/2005471)