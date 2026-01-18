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

![clash-config](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/hsalc/process_rules.png)


####    转发栈对比
Clash可选四种模式：

-   system：system 模式使用系统协议栈， 可以提供更稳定全面的 tun 体验， 且占用相对其他堆栈更低
-   gvisor：通过在用户空间中实现网络协议栈， 可以提供更高的安全性和隔离性， 同时可以避免操作系统内核和用户空间之间的切换， 从而在特定情况下具有更好的网络处理性能
-   mixed：混合堆栈， tcp 使用 system 栈， udp 使用 gvisor 栈， 使用体验可能相对更好
-   lwip：lwip 即 lightweight IP， 是一款专为嵌入式系统设计的 TCP/IP 协议栈， 采用了单线程的事件驱动模型， 性能表现可能不如 system/gvisor 协议栈（需要启用 `CGO`）

##  0x02    原理分析（Wintun）
Meta的数据流基本如下：

1.  启动TUN网卡，并设置目的IP路由（何种流量），引流到TUN网卡（`Tun-Recv`）
2.  `Tun-Process`会根据规则决定走代理（`PROXY`）OR直连模式（`DIRECT`）
3.  支持使用进程路径/进程名作为规则

![clash.meta](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/hsalc/abstract-flow-arch.png)

####    基于进程的代理配置（TCP）分析
这里以gvisor的实现为例，包含两块核心内容：
1.  在gvisor协议栈中完成TUN引流、与客户端的TCP三次握手（fake握手）等工作
2.  在经过gvisor协议栈前置处理完成之后的回调方法`handleTCPConn`中，完成规则匹配以及根据规则匹配结果进行后续Dial（直连 OR 代理）等

```GO
// handleTCPConn：处理TCP连接，参数connCtx保存了连接的上下文信息
func handleTCPConn(connCtx C.ConnContext) {
	defer func(conn net.Conn) {
		_ = conn.Close()
	}(connCtx.Conn())

	metadata := connCtx.Metadata()
	if !metadata.Valid() {
		log.Warnln("[Metadata] not valid: %#v", metadata)
		return
	}

    // ....

	// 查询当前连接connCtx的规则信息
	// 根据客户端ip/port，要连接的目的ip/port
	proxy, rule, err := resolveMetadata(connCtx, metadata)
	if err != nil {
		log.Warnln("[Metadata] parse failed: %s", err.Error())
		return
	}

    // ... 
}
```

`resolveMetadata`中会调用`match`方法，这里完成了对进程名的查找（根据源IP、源端口，协议），具体查找方式可以参考[gopsutil 使用与分析](https://pandaychen.github.io/2024/08/04/A-GOPSUTIL-STUDY/)一文，最后可以根据进程名、进程路径去搜索相关的代理规则进行匹配，从而实现了该功能

```go
// Metadata is used to store connection address
type Metadata struct {
	NetWork      NetWork    `json:"network"`
	Type         Type       `json:"type"`
	SrcIP        netip.Addr `json:"sourceIP"`
	DstIP        netip.Addr `json:"destinationIP"`
	SrcPort      uint16     `json:"sourcePort,string"`      // `,string` is used to compatible with old version json output
	DstPort      uint16     `json:"destinationPort,string"` // `,string` is used to compatible with old version json output
	InIP         netip.Addr `json:"inboundIP"`
	InPort       uint16     `json:"inboundPort,string"` // `,string` is used to compatible with old version json output
	InName       string     `json:"inboundName"`
	InUser       string     `json:"inboundUser"`
	Host         string     `json:"host"`
	DNSMode      DNSMode    `json:"dnsMode"`
	Uid          uint32     `json:"uid"`
	Process      string     `json:"process"`
	ProcessPath  string     `json:"processPath"`
	SpecialProxy string     `json:"specialProxy"`
	SpecialRules string     `json:"specialRules"`
	RemoteDst    string     `json:"remoteDestination"`
	// Only domain rule
	SniffHost string `json:"sniffHost"`
}

// 匹配规则（核心）
func match(metadata *C.Metadata) (C.Proxy, C.Rule, error) {
	configMux.RLock()
	defer configMux.RUnlock()
	var (
		resolved             bool
		attemptProcessLookup = true
	)

	if node, ok := resolver.DefaultHosts.Search(metadata.Host, false); ok {
		metadata.DstIP, _ = node.RandIP()
		resolved = true
	}

	for _, rule := range getRules(metadata) {
		if !resolved && shouldResolveIP(rule, metadata) {
			func() {
				ctx, cancel := context.WithTimeout(context.Background(), resolver.DefaultDNSTimeout)
				defer cancel()
				ip, err := resolver.ResolveIP(ctx, metadata.Host)
				if err != nil {
					log.Debugln("[DNS] resolve %s error: %s", metadata.Host, err.Error())
				} else {
					log.Debugln("[DNS] %s --> %s", metadata.Host, ip.String())
					metadata.DstIP = ip
				}
				resolved = true
			}()
		}

		// 开启了进程匹配
		if attemptProcessLookup && !findProcessMode.Off() && (findProcessMode.Always() || rule.ShouldFindProcess()) {
			attemptProcessLookup = false
			// 查找进程路径、进程名，进程id

			// 传入参数：源IP、源端口，协议
			uid, path, err := P.FindProcessName(metadata.NetWork.String(), metadata.SrcIP, int(metadata.SrcPort))
			if err != nil {
				log.Debugln("[Process] find process %s: %v", metadata.String(), err)
			} else {
				metadata.Process = filepath.Base(path) //进程名
				metadata.ProcessPath = path            //进程路径
				metadata.Uid = uid
			}
		}

		//命中规则
		if matched, ada := rule.Match(metadata); matched {
			adapter, ok := proxies[ada]
			if !ok {
				continue
			}

			// parse multi-layer nesting
			passed := false
			for adapter := adapter; adapter != nil; adapter = adapter.Unwrap(metadata, false) {
				if adapter.Type() == C.Pass {
					passed = true
					break
				}
			}
			if passed {
				log.Debugln("%s match Pass rule", adapter.Name())
				continue
			}

			if metadata.NetWork == C.UDP && !adapter.SupportUDP() {
				log.Debugln("%s UDP is not supported", adapter.Name())
				continue
			}

			return adapter, rule, nil
		}
	}

	// 默认走直连方式
	return proxies["DIRECT"], nil, nil
}
```

##  0x03    原理分析（Windivert）

TODO:gemini+图

##  0x0 参考
-   [Clash 教程](https://www.codein.icu/clashtutorial/)
-   [规则集](https://github.com/angwz/DomainRouter)
-   [[Feature] Tun #393](https://github.com/Dreamacro/clash/pull/393)
-   [虚空终端 Docs：Tun 模式](https://wiki.metacubex.one/config/inbound/tun/)
-	[net-speeder：在高延迟不稳定链路上优化单线程下载速度](https://github.com/snooda/net-speeder)