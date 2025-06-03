---
layout:     post
title:      Tracee 学习（一）：总览
subtitle:   分析一款 Linux 运行时安全及取证工具的实现
date:       2024-06-20
author:     pandaychen
catalog:    true
tags:
    - eBPF
    - Tracee
---

##  0x00    前言
Tracee 是一个用于 Linux 的运行时安全和取证工具，基于 Linux eBPF 技术在运行时跟踪系统和应用程序，并分析收集的事件以检测可疑的行为模式，整体架构如下：

![ARCH](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/tracee/architecture-1.png)

-   基于eBPF的事件采集器（Capture）：在内核中捕获事件
-   事件处理管道（Pipeline机制）：处理和丰富化事件
-   签名引擎：检测安全威胁
-   策略管理：控制事件过滤和选择

本文关联版本为[v0.23.1](https://github.com/aquasecurity/tracee/releases/tag/v0.23.1)、二进制版本[v0.23.1](https://github.com/aquasecurity/tracee/releases/download/v0.23.1/tracee-x86_64.v0.23.1.tar.gz)，运行内核版本`6.6.30-5.tl4.x86_64`

Tracee 包括两大基础组件：
-   `Trace-eBPF`：使用 eBPF 进行 Linux 追踪和取证，BPF 底层代码实现参见 tracee.bpf.c
-   `Trace-Rules`：运行时安全检测引擎，可通过管道的方式从 `Trace-eBPF` 中接受数据并进行处理

####    基础使用（采集）
1、`./tracee-ebpf  --output option:parse-arguments`，每一行代表Tracee所采集到的（一条）单一事件，其中包含下列信息：

| 表项 | 意义 | 
| :-----:| :----: | 
| TIME | 显示系统启动后事件的发生时间，单位为秒 | 
| UID | 调用进程的用户ID | 
| COMM | 调用进程名称 | 
| PID | 调用进程PID | 
| TID | 调用线程TID | 
| RET | 函数返回的值 | 
| EVENT | 事件识别符 | 
| ARGS | 提供给函数的参数列表 | 

```text
root@VM-16-15-ubuntu:~/tracee/dist# ./tracee-ebpf  --output option:parse-arguments
TIME             UID    COMM             PID     TID     RET              EVENT                     ARGS
14:20:53:862128  0      barad_agent      1209425 1593323 0                fchown                    fd: 14, owner: 0, group: 0
14:20:53:865935  0      barad_agent      1209425 1593323 0                security_inode_unlink     pathname: /usr/local/qcloud/monitor/barad/log/20240822_record.db-journal, inode: 812791, dev: 264241154, ctime: 1724307653858917003
14:20:53:867070  0      barad_agent      1209425 1593323 0                fchown                    fd: 14, owner: 0, group: 0
14:20:53:870378  0      barad_agent      1209425 1593323 0                security_inode_unlink     pathname: /usr/local/qcloud/monitor/barad/log/20240822_record.db-journal, inode: 812791, dev: 264241154, ctime: 1724307653862917021
14:20:53:871062  0      barad_agent      1209425 1593323 0                fchown                    fd: 14, owner: 0, group: 0
14:20:53:874137  0      barad_agent      1209425 1593323 0                security_inode_unlink     pathname: /usr/local/qcloud/monitor/barad/log/20240822_record.db-journal, inode: 812791, dev: 264241154, ctime: 1724307653866917041
14:20:53:874842  0      barad_agent      1209425 1593323 0                fchown                    fd: 14, owner: 0, group: 0
14:20:53:878247  0      barad_agent      1209425 1593323 0                security_inode_unlink     pathname: /usr/local/qcloud/monitor/barad/log/20240822_record.db-journal, inode: 812791, dev: 264241154, ctime: 1724307653870917060
14:20:54:746947  0      idsagent      1189601 1189632 0                security_socket_connect   sockfd: 8, type: SOCK_DGRAM, remote_addr: map[sa_family:AF_INET sin_addr:127.0.0.53 sin_port:53]
14:20:54:747032  0      idsagent      1189601 1189632 0                net_packet_dns_request    metadata: 127.0.0.1 127.0.0.53 53167 53 17 73 any, dns_questions: [grpc.ids.store AAAA IN]
14:20:54:746947  0      idsagent      1189601 1189606 0                security_socket_connect   sockfd: 7, type: SOCK_DGRAM, remote_addr: map[sa_family:AF_INET sin_addr:127.0.0.53 sin_port:53]
14:20:54:747032  0      idsagent      1189601 1189606 0                net_packet_dns_request    metadata: 127.0.0.1 127.0.0.53 38957 53 17 73 any, dns_questions: [grpc.ids.store A IN]
14:20:54:747706  102    systemd-resolve  1372634 1372634 0                net_packet_dns_response   metadata: 127.0.0.53 127.0.0.1 53 38957 17 89 any, dns_response: [grpc.ids.store A IN [A 560 127.0.0.1]]
14:20:54:747721  102    systemd-resolve  1372634 1372634 0                net_packet_dns_response   metadata: 127.0.0.53 127.0.0.1 53 38957 17 89 any, dns_response: [grpc.ids.store A IN [A 560 127.0.0.1]]
14:20:54:748012  102    systemd-resolve  1372634 1372634 0                security_socket_connect   sockfd: 11, type: SOCK_DGRAM, remote_addr: map[sa_family:AF_INET sin_addr:183.60.83.19 sin_port:53]
14:20:54:748183  102    systemd-resolve  1372634 1372634 0                net_packet_dns_request    metadata: 10.206.16.15 183.60.83.19 33864 53 17 62 any, dns_questions: [grpc.ids.store AAAA IN]
14:20:54:749719  0      swapper/1        0       0       0                net_packet_dns_response   metadata: 183.60.83.19 10.206.16.15 53 33864 17 126 any, dns_response: [grpc.ids.store AAAA IN []]
14:20:54:749932  102    systemd-resolve  1372634 1372634 0                net_packet_dns_response   metadata: 127.0.0.53 127.0.0.1 53 53167 17 137 any, dns_response: [grpc.ids.store AAAA IN []]
14:20:54:749952  102    systemd-resolve  1372634 1372634 0                net_packet_dns_response   metadata: 127.0.0.53 127.0.0.1 53 53167 17 137 any, dns_response: [grpc.ids.store AAAA IN []]
14:20:54:750033  0      idsagent      1189601 1189604 0                security_socket_connect   sockfd: 7, type: SOCK_STREAM, remote_addr: map[sa_family:AF_INET sin_addr:127.0.0.1 sin_port:8000]
14:20:54:865306  0      sh               1593326 1593326 0                sched_process_exec        cmdpath: /bin/sh, pathname: /usr/bin/dash, dev: 264241154, inode: 601, ctime: 1650801788632000000, inode_mode: 33261, interpreter_pathname: /usr/lib/x86_64-linux-gnu/ld-linux-x86-64.so.2, interpreter_dev: 264241154, interpreter_inode: 83, interpreter_ctime: 1717491233490378804, argv: [/bin/sh -c cat /proc/meminfo |grep 'HardwareCorrupted' |awk 'print $2'], interp: /bin/sh, stdin_type: S_IFCHR, stdin_path: /dev/null, invoked_from_kernel: 0, env: <nil>
14:20:54:867010  0      awk              1593329 1593329 0                sched_process_exec        cmdpath: /usr/bin/awk, pathname: /usr/bin/gawk, dev: 264241154, inode: 394, ctime: 1696647869063262130, inode_mode: 33261, interpreter_pathname: /usr/lib/x86_64-linux-gnu/ld-linux-x86-64.so.2, interpreter_dev: 264241154, interpreter_inode: 83, interpreter_ctime: 1717491233490378804, argv: [awk print $2], interp: /usr/bin/awk, stdin_type: S_IFIFO, stdin_path: , invoked_from_kernel: 0, env: <nil>
14:20:54:868613  0      grep             1593328 1593328 0                sched_process_exec        cmdpath: /usr/bin/grep, pathname: /usr/bin/grep, dev: 264241154, inode: 709, ctime: 1650801789108000000, inode_mode: 33261, interpreter_pathname: /usr/lib/x86_64-linux-gnu/ld-linux-x86-64.so.2, interpreter_dev: 264241154, interpreter_inode: 83, interpreter_ctime: 1717491233490378804, argv: [grep HardwareCorrupted], interp: /usr/bin/grep, stdin_type: S_IFIFO, stdin_path: , invoked_from_kernel: 0, env: <nil>
14:20:54:869617  0      cat              1593327 1593327 0                sched_process_exec        cmdpath: /usr/bin/cat, pathname: /usr/bin/cat, dev: 264241154, inode: 558, ctime: 1650801788564000000, inode_mode: 33261, interpreter_pathname: /usr/lib/x86_64-linux-gnu/ld-linux-x86-64.so.2, interpreter_dev: 264241154, interpreter_inode: 83, interpreter_ctime: 1717491233490378804, argv: [cat /proc/meminfo], interp: /usr/bin/cat, stdin_type: S_IFCHR, stdin_path: /dev/null, invoked_from_kernel: 0, env: <nil>
14:20:54:966874  0      barad_agent      1209425 1593324 0                security_socket_connect   sockfd: 13, type: SOCK_DGRAM, remote_addr: map[sa_family:AF_INET sin_addr:172.0.0.1 sin_port:80]
14:20:54:968809  0      barad_agent      1209424 1209424 0                security_socket_connect   sockfd: 13, type: SOCK_STREAM, remote_addr: map[sa_family:AF_INET sin_addr:169.254.0.4 sin_port:80]
14:20:54:972065  0      barad_agent      1209424 1209424 0                net_packet_http_request   metadata: 10.206.16.15 169.254.0.4 57850 80 6 221 any, http_request: POST HTTP/1.1 169.254.0.4 /ca_report.cgi map[Accept-Encoding:[identity] Connection:[close] Content-Length:[796] Content-Type:[application/json] User-Agent:[Python-urllib/2.6]] 796
14:20:55:099351  0      swapper/1        0       0       0                net_packet_http_response  metadata: 169.254.0.4 10.206.16.15 80 57850 6 256 any, http_response: 200 OK 200 HTTP/1.1 map[Content-Length:[74] Content-Type:[application/json; charset=utf-8] Date:[Thu, 22 Aug 2024 06:20:55 GMT]] 74
```

####    支持的 event
参考 [Events](https://aquasecurity.github.io/tracee/dev/docs/events/)，包含了如下（其中 syscalls 是笔者关心的部分）：

-   Security Events
-   Network Events
-   Extra Events
-   [Syscalls](https://aquasecurity.github.io/tracee/dev/docs/events/builtin/syscalls/)

##  0x01    工作原理

####    tracee架构
![arch](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/tracee/architecture.png)

如上图（图来自于版本v0.8.3），一共拆分为五个阶段：

1、阶段一，内核态eBPF采集器产生到用户态（Userland）的Tracee事件，包含Tracepoints、Probes以及Trafic Control Hooks

2、阶段二，用户态对内核态的事件进行丰富（ENRICHED）及补全，包含如下

-   内核事件（Syscalls，TracePoints，kprobes）
-   操作系统事件（Running Containers以及其他）
-   派生（来自其他）事件
-   网络事件

3、阶段三，基于已有的规则签名（Signatures）进行检测，签名[支持](https://aquasecurity.github.io/tracee/v0.8.3/detecting/#selecting-signatures)

4、阶段四，检测事件的消费及后续处理

-   捕获事件或者二进制

####    采集器

Tracee 中的 `tracee-ebpf` 模块的核心能力包括： 事件跟踪（trace）、抓取（capture）和输出（output），`tracee-ebpf` 的核心能力在于底层 eBPF 机制抓取事件的能力，`tracee-ebpf` 默认实现了诸多的事件抓取功能，可以通过 `trace-ebpf -l` 参看到底层支持的函数全集（本版本约`1800+`）

```text
root@VM-16-15-ubuntu:~/tracee/dist# ./tracee-ebpf -l|wc -l
1838

Syscall Events

+------------------------------+--------------------------------+------------------------------------------+
| RULE                         | SETS                           | ARGUMENTS                                |
+------------------------------+--------------------------------+------------------------------------------+
| read                         | syscalls, fs, fs_read_write    | int fd, void* buf, size_t                |
|                              |                                | count                                    |
+------------------------------+--------------------------------+------------------------------------------+
| write                        | syscalls, fs, fs_read_write    | int fd, void* buf, size_t                |
|                              |                                | count                                    |
+------------------------------+--------------------------------+------------------------------------------+
| open                         | syscalls, fs, fs_file_ops      | const char* pathname, int                |
|                              |                                | flags, mode_t mode                       |
+------------------------------+--------------------------------+------------------------------------------+
| close                        | syscalls, fs, fs_file_ops      | int fd                                   |
+------------------------------+--------------------------------+------------------------------------------+
| stat                         | syscalls, fs, fs_file_attr     | const char* pathname, struct             |
|                              |                                | stat* statbuf                            |
+------------------------------+--------------------------------+------------------------------------------+
| fstat                        | syscalls, fs, fs_file_attr     | int fd, struct stat* statbuf             |
+------------------------------+--------------------------------+------------------------------------------+
| lstat                        | syscalls, fs, fs_file_attr     | const char* pathname, struct             |
|                              |                                | stat* statbuf                            |
+------------------------------+--------------------------------+------------------------------------------+
```

-   `RULE`：系统调用函数
-   `SETS`：该函数归属为的子类（注可归属多个，比如 `read` 函数，归属于 `syscalls`/`fs`/`fs_read_write` 3 个子类，除了 `fs` 外，`net` 集合中也包含了许多的跟踪函数）
-   `ARGUMENTS`：为该函数的原型，可以使用参数中的字段进行过滤，支持特定的运算，比如 `==`、 `!=` 等逻辑操作符，对于字符串也支持通配符操作

例如：

```bash
--trace s=fs --trace e!=open,openat 跟踪 fs 集合中的所有事件，但是不包括 open,openat 两个函数
--trace openat.pathname!=/tmp/1,/bin/ls 这里表示不跟踪 openat 事件中，pathname 为 /tmp/1,/bin/ls 的事件，注意这里的 openat.pathname 为跟踪函数名与函数参数的组合
```

以上跟踪事件的过滤条件会**通过接口设置进内核中对应的 map 结构中，在完成过滤和事件跟踪以后，通过 perf_event 的方式上报到用户空间程序中，可以保存到文件后续进行处理，或者直接通过管道发送至 `tracee-rule` 进行解析和进行更高级别的上报**

####  Tracee Pipeline Concept
![pipeline](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/tracee/tracee-pipeline-overview.png)

Pipeline的作用过程如下：

1.  每个CPU不断从内核运行的eBPF程序生成事件
2.  每当触发其附加的内核hooks时，都会执行EBPF程序
3.  eBPF程序基于given filters决定是否应该将事件提交给 tracee-ebpf
4.  这些事件被发送到 libbpfgo 通过perfbuffer（共享内存环缓冲区）传递到用户态
5.  libbpfgo 将内核eBPF采集器收集的事件发送给Tracee golang channels
6.  tracee-ebpf 收到事件后，解析及转换参数类型的事件、丰富事件，如果需要捕获相关事件则进一步处理
7.  tracee-ebpf 通过printer机制将事件发送给 Tracee-rules 模块
8.  tracee-rules 接受这些事件并使用签名进行风险评估（推荐基于golang的规则）
9.  Tracee-rules 给出风险评估结果

####    风险检测

| 风险 | 说明 | 应用范围 |
| :-----| :---- | :----|
| Standard Input/Output Over Socket | 	将进程的标准输入/输出重定向到套接字 | CVM/容器 |
| Anti-Debugging | 进程使用反调试技术来阻止调试器 | CVM/容器 |
| Code injection | 将代码注入另一个进程 | CVM/容器 |
| Dynamic Code Loading | 写入可执行分配的内存区域 | CVM/容器 |
| Fileless Execution | 直接从内存中运行进程（不使用磁盘文件） | CVM/容器 |
| kernel module loading | 尝试加载内核模块检测 | CVM/容器 |
| LD_PRELOAD | 使用 LD_PRELOAD 来允许进程hook | CVM/容器 |
| Container Host Mount | 将主机文件系统挂载到容器中 | 容器 |
| Dropped Executable | 在运行时从容器中创建或删除可执行文件 | CVM/容器 |
| Illegitimate Shell | 生成 Shell 程序 | CVM/容器 |
| K8S API Connection | 连接到 Kubernetes 集群 API 服务器 | 容器 |
| K8S Service Account Use | 	读取容器中的 Kubernetes 服务帐户令牌文件 | 容器 |
| K8S TLS Certificate Theft | 访问用于 Kubernetes 组件之间安全通信的 TLS 证书	 | CVM/容器 |


举例来说，选用Fileless Execution（直接从内存执行进程，磁盘中无文件）行为，典型工具为`elfexec`进行验证：

```BASH
wget https://github.com/abbat/elfexec/releases/download/v0.3/elfexec.x64.glibc.xz
xz -d elfexec.x64.glibc.xz
chmod u+x elfexec.x64.glibc && mv ./elfexec.x64.glibc ./elfexec

echo '
#include <unistd.h>

int main(int argc, char* argv[])
{
    write(STDOUT_FILENO, "Hello!\n", 7);
    return 0;
}
' | cc -xc - -o /dev/stdout | elfexec
```

tracee-rules捕捉到结果如下，触发的检测签名为`TRC-5`

```TEXT
Loaded 14 signature(s): [TRC-1 TRC-13  TRC-2 TRC-3 TRC-11 TRC-9 TRC-4 TRC-5 TRC-12 TRC-8 TRC-6 TRC-10 TRC-7]

*** Detection ***
Time: xxxxxx
Signature ID: TRC-5
Signature: Fileless Execution
Data: map[]
Command: elfexec
Hostname: localhost
```

####    签名引擎

2、自定义规则，`tracee-rules`提供了两种方式自定义规则，一是使用`.rego`语言的规则文本，而是使用go Signature接口的规则

-   [rego规则](https://github.com/aquasecurity/tracee/tree/v0.8.3/signatures/rego)：基于文本的规则参考，适用`v0.8.3`版本
-   [基于golang的规则](https://github.com/aquasecurity/tracee/tree/v0.8.3/signatures/golang)

##  0x02  内核态开发
笔者感兴趣的内核态hook实现集中在此[`tracee.bpf.c`](https://github.com/aquasecurity/tracee/blob/main/pkg/ebpf/c/tracee.bpf.c)，除此之外：

-   MAPS[定义](https://github.com/aquasecurity/tracee/blob/main/pkg/ebpf/c/maps.h)
-   kernel [封装函数](https://github.com/aquasecurity/tracee/blob/main/pkg/ebpf/c/tracee.h)
-   [通用封装函数](https://github.com/aquasecurity/tracee/tree/main/pkg/ebpf/c/common)
-   [通用结构定义](https://github.com/aquasecurity/tracee/blob/main/pkg/ebpf/c/types.h)


##  0x03    参考
-   [eBPF 安全项目 Tracee 初探](https://www.do1618.com/archives/1702/ebpf-an-quan-xiang-mu-tracee-chu-tan/)
-   [Tracee Architecture Overview](https://aquasecurity.github.io/tracee/v0.8.0/architecture/)
-   [Tracee 性能](https://aquasecurity.github.io/tracee/v0.8.0/deep-dive/performance/)
-   [tracee 源码初探 (一)](https://www.cnblogs.com/janeysj/p/16254048.html)
-   [libsinsp, libscap, the kernel module driver, and the eBPF driver sources](https://github.com/falcosecurity/libs)
-   [execveat：doc](https://aquasecurity.github.io/tracee/v0.22/docs/events/builtin/syscalls/execveat/)
-   [使用tracee编写规则追踪系统安全事件](https://www.freebuf.com/articles/system/304038.html)