---
layout:     post
title:      WinTun && WinDivert 使用与分析
subtitle:   基于Clash等开源项目的一些技术实现总结
date:       2024-06-01
author:     pandaychen
catalog:    true
tags:
    - Clash
---


##  0x00 前言
本文汇总下笔者在研究Clash（ClashMeta）等一些windows上的透明代理项目的一些技术思路总结，基于wintun OR windivert

##  0x01    配置&&功能

####    进程过滤
[Rules 规则](https://clash.wiki/configuration/rules.html#process-name-%E6%BA%90%E8%BF%9B%E7%A8%8B%E5%90%8D)可支持按照PROCESS-NAME 源进程名、PROCESS-PATH 源进程路径等走对应的规则，比如

```bash
# clash 开 tun 模式代理，根据进程名匹配规则排除游戏代理 Netch.exe
- PROCESS-NAME,Netch.exe,DIRECT
```

```bash
PROCESS-NAME 源进程名
PROCESS-NAME 规则用于根据发送数据包的进程名称路由数据包

#目前, 仅支持 macOS、Linux、FreeBSD 和 Windows

PROCESS-NAME,nc,DIRECT 将任何来自进程 nc 的数据包路由到 DIRECT

PROCESS-PATH 源进程路径
PROCESS-PATH 规则用于根据发送数据包的进程路径路由数据包

#目前, 仅支持 macOS、Linux、FreeBSD 和 Windows

PROCESS-PATH,/usr/local/bin/nc,DIRECT 将任何来自路径为 /usr/local/bin/nc 的进程的数据包路由到 DIRECT
```


####    转发栈对比
Clash可选四种模式：

-   system：system 模式使用系统协议栈， 可以提供更稳定全面的 tun 体验， 且占用相对其他堆栈更低
-   gvisor：通过在用户空间中实现网络协议栈， 可以提供更高的安全性和隔离性， 同时可以避免操作系统内核和用户空间之间的切换， 从而在特定情况下具有更好的网络处理性能
-   mixed：混合堆栈， tcp 使用 system 栈， udp 使用 gvisor 栈， 使用体验可能相对更好
-   lwip：lwip 即 lightweight IP， 是一款专为嵌入式系统设计的 TCP/IP 协议栈， 采用了单线程的事件驱动模型， 性能表现可能不如 system/gvisor 协议栈（需要启用 `CGO`）

##  0x02 参考
-   [Clash 教程](https://www.codein.icu/clashtutorial/)
-   [规则集](https://github.com/angwz/DomainRouter)
-   [[Feature] Tun #393](https://github.com/Dreamacro/clash/pull/393)
-   [虚空终端 Docs：Tun 模式](https://wiki.metacubex.one/config/inbound/tun/)
