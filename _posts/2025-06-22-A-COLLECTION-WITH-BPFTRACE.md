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


##  0x01    进程类


##  0x02    网络类


##  0x03    VFS类


##  0x04    内存类


##  0x05    其他

1、`sys_enter_finit_module`

```BASH
#!/usr/bin/env bpftrace

BEGIN {
    printf("Tracing init_module and finit_module syscalls... Hit Ctrl+C to stop.\n");
}

tracepoint:syscalls:sys_enter_finit_module {
    printf("Syscall executed: %s (PID: %d, UID: %d, Command: %s),umod: %d\n", probe, pid, uid, comm, args->fd);
}
```


##  0x0 参考
-   []()