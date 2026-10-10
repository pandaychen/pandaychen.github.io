---
layout:     post
title:      Golang 中网络编程开源项目汇总（持续更新）
subtitle:   可以从开源项目中学习到什么（二）
date:       2020-03-08
author:     pandaychen
catalog:    true
header-img: img/panda-md-pic7.jpg
tags:
    - Golang
    - OpenSource
---

## 0x00 前言

本文汇总了 Golang 网络编程领域值得学习和参考的开源项目，按功能分类整理，持续更新中。这些项目涵盖了 TCP 框架、高性能 Web 服务器、RPC 框架、代理隧道、网络协议实现、IM 长连接、负载均衡、DNS、VPN 组网、WebSocket、网络工具等方向，适合作为 Go 网络编程的学习和实战参考

## 0x01 TCP 网络框架

TCP 网络框架是 Go 网络编程的基石，下面列出的项目从不同角度（Reactor / Proactor / Goroutine-per-Connection）实现了高性能网络层抽象：

-   [Zinx](https://github.com/aceld/zinx)：基于 Golang 轻量级 TCP 并发服务器框架，采用 Goroutine-per-Connection 模型，适合学习 Go 网络编程入门
-   [link](https://github.com/funny/link)：Go 语言网络层脚手架，设计简洁，支持自定义协议编解码
-   [gnet](https://github.com/panjf2000/gnet)：高性能、轻量级、非阻塞的事件驱动 Go 网络框架，基于 epoll/kqueue 实现 Reactor 模型，性能显著优于标准库 `net`
-   [evio](https://github.com/tidwall/evio)：快速的事件循环网络框架，基于 epoll/kqueue，API 极简，适合理解 event-loop 模型在 Go 中的实现
-   [nbio](https://github.com/lesismal/nbio)：高性能非阻塞 I/O 网络框架，同时支持 TCP/UDP/Unix Socket/TLS/HTTP/WebSocket，提供标准库兼容的 API
-   [gaio](https://github.com/xtaci/gaio)：基于 Proactor 模型的高性能异步网络 I/O 库，通过 io_uring/epoll 实现，设计理念独特
-   [gotcp](https://github.com/gansidui/gotcp)：轻量级 TCP 服务器框架，代码精简，适合学习 Go TCP 服务端的基本组织方式
-   [easytcp](https://github.com/DarthPestilane/easytcp)：优雅的 Go TCP 框架，支持路由、中间件、编解码器，设计类 Web 框架风格
-   [netpoll](https://github.com/cloudwego/netpoll)：字节跳动开源的高性能非阻塞 I/O 网络库，CloudWeGo 体系的网络层基座，支持 epoll/kqueue + 零拷贝

## 0x02 Web 框架 / HTTP 服务器

-   [WSP](https://github.com/simplejia/wsp)：Golang Web Server Framework，简洁灵活
-   [fasthttp](https://github.com/valyala/fasthttp)：高性能 HTTP 服务器和客户端库，比标准库 `net/http` 快约 10 倍，零内存分配设计，适合高并发场景
-   [Gin](https://github.com/gin-gonic/gin)：Go 生态最流行的 HTTP Web 框架，基于 httprouter，支持中间件、JSON 校验、路由分组，API 开发首选
-   [Fiber](https://github.com/gofiber/fiber)：受 Express.js 启发、基于 fasthttp 的 Web 框架，API 风格对 Node.js 开发者友好，性能优秀
-   [Echo](https://github.com/labstack/echo)：高性能、极简 Go Web 框架，支持自动 TLS、HTTP/2、中间件链，适合 RESTful API 开发
-   [Chi](https://github.com/go-chi/chi)：轻量级、符合 Go 标准库 `net/http` 风格的路由器，支持中间件、路由分组，无外部依赖
-   [Caddy](https://github.com/caddyserver/caddy)：Go 编写的现代化 Web 服务器，自动 HTTPS（内置 ACME），模块化架构，生产级可用

## 0x03 RPC 框架

-   [gRPC-Go](https://github.com/grpc/grpc-go)：Google 官方的 Go 语言 gRPC 实现，基于 HTTP/2 + Protobuf，支持流式 RPC、拦截器、负载均衡，微服务通信事实标准
-   [rpcx](https://github.com/smallnest/rpcx)：类似 Dubbo 的高性能 Go RPC 框架，支持多种注册中心（ZooKeeper/Etcd/Consul）、多种序列化协议、服务治理完备
-   [Kitex](https://github.com/cloudwego/kitex)：字节跳动开源的高性能 Go RPC 框架，基于 Thrift/Protobuf，内置服务治理能力，与 netpoll 深度集成
-   [go-micro](https://github.com/go-micro/go-micro)：分布式微服务框架，抽象了服务发现、负载均衡、消息编码、RPC 通信等，可插拔架构
-   [Dragonboat](https://github.com/lni/dragonboat)：高性能多组 Raft 共识库，纯 Go 实现，可作为分布式系统的基础构建块
-   [Twirp](https://github.com/twitchtv/twirp)：Twitch 开源的简单 RPC 框架，基于 Protobuf，同时生成 HTTP 1.1 JSON 和 Protobuf 接口，比 gRPC 更轻量

## 0x04 代理 / 隧道 / 网关

-   [frp](https://github.com/fatedier/frp)：高性能反向代理和内网穿透工具，支持 TCP/UDP/HTTP/HTTPS，客户端-服务端架构，NAT 穿透首选方案
-   [gost](https://github.com/ginuerzh/gost)：GO Simple Tunnel，功能强大的安全隧道工具，支持多种代理协议（HTTP/SOCKS5/Shadowsocks）和传输方式（TLS/WebSocket/KCP/QUIC）的链式组合
-   [chisel](https://github.com/jpillora/chisel)：基于 HTTP + WebSocket 的 TCP/UDP 隧道，类似 SSH 端口转发，单二进制文件部署
-   [rathole](https://github.com/rapiz1/rathole)：轻量级安全的 NAT 穿透工具（Rust 实现，但设计思想值得参考），与 frp 类似但更轻量
-   [Traefik](https://github.com/traefik/traefik)：云原生反向代理和负载均衡器，自动服务发现（Docker/K8s/Consul），支持动态配置、Let's Encrypt 自动 HTTPS
-   [APISIX](https://github.com/apache/apisix)：Apache 高性能云原生 API 网关，基于 OpenResty/Lua，但其 Go 插件机制（go-plugin-runner）值得学习
-   [easegress](https://github.com/megaease/easegress)：MegaEase 开源的云原生流量编排系统，支持 API 网关、流量调度、服务网格 Sidecar 等场景

## 0x05 网络协议实现

-   [quic-go](https://github.com/quic-go/quic-go)：QUIC 协议的纯 Go 实现，支持 HTTP/3，是学习 QUIC 协议和 Go 网络编程的绝佳项目
-   [kcp-go](https://github.com/xtaci/kcp-go)：KCP 协议的 Go 实现，一种快速可靠的 ARQ 协议，以带宽换延迟，适合实时性要求高的场景（游戏、视频通话）
-   [kcptun](https://github.com/xtaci/kcptun)：基于 KCP 协议的稳定安全隧道，将 KCP+FEC+加密 组合，提供低延迟的数据传输通道
-   [smux](https://github.com/xtaci/smux)：简洁高效的流多路复用库（Stream Multiplexer），在单条 TCP 连接上实现多条逻辑流，代码精炼适合学习
-   [yamux](https://github.com/hashicorp/yamux)：HashiCorp 的流多路复用实现，Consul/Nomad 等项目的底层网络通信组件
-   [cmux](https://github.com/soheilhy/cmux)：连接多路复用器，可在同一端口上同时 serve gRPC、HTTP、SSH 等不同协议，通过读取连接头部字节来分发
-   [water](https://github.com/songgao/water)：Go 语言 TUN/TAP 虚拟网络设备库，是构建 VPN、网络隧道等应用的基础组件

## 0x06 PUSH / IM 长连接

-   [connsvr](https://github.com/simplejia/connsvr)：Golang 实现的长连接服务，适合群推，比如推消息、推比赛比分等，功能丰富
-   [goim](https://github.com/Terry-Mao/goim)：高性能 IM 和推送通知服务器集群，B 站开源，经过大规模生产验证，支持百万级长连接
-   [Open-IM-Server](https://github.com/openimsdk/open-im-server)：开源高性能 IM 服务端，支持单聊/群聊/消息漫游/离线推送，全套 IM 解决方案
-   [Centrifugo](https://github.com/centrifugal/centrifugo)：可扩展的实时消息服务器，支持 WebSocket/SSE/HTTP-streaming，与任意后端语言集成，适合实时通知/聊天/协作
-   [Mercury](https://github.com/marmotedu/mercury-im)：Go 实现的即时通讯系统，架构清晰可作学习参考
-   [Commet](https://github.com/panjf2000/gnet)（gnet 之上的应用层）：展示了如何用高性能网络库构建推送服务

## 0x07 负载均衡

-   [GOLB](https://github.com/onestraw/golb)：Go 实现的简洁 L7 负载均衡器，支持轮询、加权轮询、IP Hash 等策略，代码量适中适合学习
-   [gobetween](https://github.com/yyyar/gobetween)：Go 实现的高性能 L4 负载均衡器，支持 TCP/UDP/TLS，内置健康检查和多种发现机制
-   [Traefik](https://github.com/traefik/traefik)：（见代理/隧道/网关章节）同时也是优秀的负载均衡器，支持多种策略和自动服务发现
-   [Caddy](https://github.com/caddyserver/caddy)：（见 Web 框架章节）内置负载均衡、健康检查、反向代理功能

## 0x08 DNS

-   [CoreDNS](https://github.com/coredns/coredns)：CNCF 毕业项目，Kubernetes 默认 DNS 服务器，插件式架构，可灵活扩展，是学习 Go 实现 DNS 服务的最佳参考
-   [dnscrypt-proxy](https://github.com/DNSCrypt/dnscrypt-proxy)：灵活的 DNS 代理，支持 DNS-over-HTTPS（DoH）、DNS-over-TLS（DoT）、DNSCrypt 等加密 DNS 协议

## 0x09 VPN / 组网

-   [WireGuard-Go](https://github.com/WireGuard/wireguard-go)：WireGuard VPN 的纯 Go 用户态实现，代码精炼，是学习 VPN 协议和 TUN 设备编程的绝佳项目
-   [Nebula](https://github.com/slackhq/nebula)：Slack 开源的可扩展 overlay 组网工具，基于 Noise 协议，支持 NAT 穿透和证书认证，适合构建分布式 P2P 网络
-   [Tailscale](https://github.com/tailscale/tailscale)：基于 WireGuard 的零配置 VPN，提供 mesh 组网能力，自动 NAT 穿透，企业级可用
-   [gVisor-netstack](https://github.com/google/gvisor/tree/master/pkg/tcpip)：Google gVisor 的用户态 TCP/IP 协议栈实现，纯 Go 编写，是学习网络协议栈实现的宝藏项目

## 0x0A WebSocket

-   [gorilla/websocket](https://github.com/gorilla/websocket)：Go 生态最成熟的 WebSocket 库，API 完善、性能优秀，支持压缩和并发安全
-   [nhooyr/websocket](https://github.com/nhooyr/websocket)：现代化的 WebSocket 库，API 更简洁，支持 `context.Context`，并发安全设计
-   [Melody](https://github.com/olahol/melody)：基于 gorilla/websocket 的极简 WebSocket 框架，适合快速构建 WebSocket 服务
-   [gobwas/ws](https://github.com/gobwas/ws)：零分配的 WebSocket 库，适合高并发百万连接场景，不使用 goroutine-per-connection，内存占用极低

## 0x0B 网络工具 / 测试

-   [gopacket](https://github.com/google/gopacket)：Google 维护的 Go 抓包库，封装 libpcap，支持 BPF 过滤和协议解码，网络分析利器
-   [hey](https://github.com/rakyll/hey)：轻量级 HTTP 压测工具（ab 的替代品），支持 HTTP/2，Google 出品
-   [vegeta](https://github.com/tsenart/vegeta)：HTTP 负载测试工具和库，支持恒定 QPS 模式，可编程使用，输出格式丰富
-   [httpstat](https://github.com/davecheney/httpstat)：可视化 HTTP 请求各阶段耗时（DNS/TCP/TLS/Server/Transfer），类似 curl 的增强版
-   [tcping](https://github.com/cloverstd/tcping)：Go 实现的 TCP Ping 工具，支持 HTTP/HTTPS 模式，用于检测端口连通性和延迟

## 0x0C Service Mesh / 数据面

-   [MOSN](https://github.com/mosn/mosn)：蚂蚁集团开源的云原生网络代理，模块化、可扩展，支持 xDS API，可作为 Service Mesh 数据面
-   [Istio](https://github.com/istio/istio)：最流行的 Service Mesh 控制面，Go 实现，管理微服务间的流量、安全和可观测性
-   [Envoy Gateway](https://github.com/envoyproxy/gateway)：Envoy 的 Kubernetes Gateway API 实现，控制面用 Go 编写

## 0x0D Nginx Modules

-   [Nginx 正向代理](https://github.com/chobits/ngx_http_proxy_connect_module)：让 Nginx 支持 HTTP CONNECT 方法的正向代理模块

## 0x0E 推荐项目总结

| 分类 | 项目 | 核心亮点 | 学习价值 |
|------|------|---------|---------|
| **TCP 框架** | gnet | epoll/kqueue Reactor 模型，性能极致 | 理解 Go 中非标准库网络模型的实现 |
| **TCP 框架** | netpoll | 字节跳动出品，CloudWeGo 网络基座 | 工业级非阻塞 I/O 实现 |
| **TCP 框架** | evio | 极简 event-loop，代码量少 | 快速理解事件驱动网络编程 |
| **Web 框架** | Gin | 生态最全，路由性能优异 | Go Web 开发标准参考 |
| **Web 框架** | fasthttp | 零分配设计，性能 10x net/http | 理解高性能 HTTP 实现技巧 |
| **RPC 框架** | gRPC-Go | HTTP/2 + Protobuf，生态成熟 | 微服务 RPC 通信标准实现 |
| **RPC 框架** | Kitex | 字节跳动内部大规模验证 | 学习高性能 RPC + 服务治理一体化设计 |
| **代理隧道** | frp | 内网穿透首选，功能全面 | 理解反向代理和 NAT 穿透原理 |
| **代理隧道** | gost | 协议组合链式代理，灵活度极高 | 多层隧道协议编排的设计思想 |
| **协议实现** | quic-go | QUIC/HTTP3 纯 Go 实现 | 下一代传输层协议的工程实现 |
| **协议实现** | smux | 精炼的流多路复用实现 | 理解连接多路复用的核心原理 |
| **协议实现** | cmux | 同端口多协议复用 | 巧妙的协议嗅探和分发设计 |
| **IM 长连接** | goim | B 站开源，支持百万级连接 | 大规模 IM 系统架构设计 |
| **IM 长连接** | Centrifugo | 实时消息服务，协议丰富 | 通用实时消息中间件架构 |
| **DNS** | CoreDNS | K8s 默认 DNS，CNCF 毕业项目 | 插件式 DNS 服务器设计 |
| **VPN 组网** | WireGuard-Go | VPN 用户态实现，代码精简 | TUN 设备编程 + VPN 协议栈 |
| **VPN 组网** | Nebula | Slack 出品，P2P overlay 组网 | 分布式组网 + NAT 穿透技术 |
| **WebSocket** | gobwas/ws | 零分配，百万连接级别 | 极致性能的 WebSocket 实现 |
| **网络工具** | gopacket | libpcap 封装，协议解码 | 网络抓包和协议分析 |
| **Service Mesh** | MOSN | 蚂蚁出品，模块化数据面 | 理解 Service Mesh 数据面实现 |
| **负载均衡** | gobetween | L4 负载均衡，多种发现机制 | LB 核心算法和健康检查实现 |

**推荐学习路径：**

- 入门阶段：Zinx（TCP 基础） → Gin（Web 开发） → gorilla/websocket（长连接）
- 进阶阶段：gnet/evio（事件驱动模型） → smux/cmux（多路复用） → gRPC-Go（RPC 框架）
- 高阶阶段：quic-go（新协议实现） → netpoll + Kitex（工业级框架） → WireGuard-Go（VPN 协议栈）
- 架构阶段：goim（大规模 IM） → CoreDNS（插件化架构） → MOSN/Traefik（云原生网络基础设施）

转载请注明出处，本文采用 [CC4.0](http://creativecommons.org/licenses/by-nc-nd/4.0/) 协议授权
