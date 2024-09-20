---
layout:     post
title:      主机入侵检测系统 Elkeid：设计与分析（一）
subtitle:   后台服务模块
date:       2024-08-19
author:     pandaychen
catalog:    true
tags:
    - HIDS
    - ELKEID
---


##  0x00    前言
Elkeid 后台主要模块如下：

-   AgentCenter 模块：负责与 Agent 进行通信，采集 Agent 数据并简单处理后汇总到消息队列集群，同时也负责对 Agent 进行管理包括 Agent 的升级，配置修改，任务下发等。同时 AgentCenter 也对外提供 HTTP 接口，Manager 模块通过这些 HTTP 接口实现对 AgentCenter 和 Agent 的管理和监控
-   ServiceDiscovery 模块：注册中心，后台中的各个服务模块都需要向 ServiceDiscovery 定时注册、同步服务信息，从而保证各个服务模块中的实例相互可见，便于直接通信。由于 ServiceDiscovery 模块维护了各个注册服务的状态信息，所以当服务使用方在请求服务发现时，ServiceDiscovery 模块会进行负载均衡。比如 Agent 请求获取 AgentCenter 模块实例列表，ServiceDiscovery 模块直接返回负载压力最小的 AgentCenter 实例
-   Manager 模块：负责对整个后台进行管理并提供相关的查询、管理接口。包括管理 AgentCenter 集群，监控 AgentCenter 状态，控制 AgentCenter 服务相关参数，并通过 AgentCenter 管理所有的 Agent，收集 Agent 运行状态，向 Agent 下发任务（走控制 control 信道），同时 manager 模块也负责管理实时和离线计算集群
-   Elkeid Console：Elkeid 前端部分，通过 Elkeid Console 可查看告警 / 资产数据
-   Elkeid HUB：Elkeid HIDS RuleEngine，对数据进行分析和检测（暂未开源）

本文基于 [v1.9.1.9](https://github.com/bytedance/Elkeid/releases/tag/v1.9.1.9) 版本分析，本文主要关注如下模块的实现

![backend](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/elkeid/arch/agent2agentcenter.png)


##  0x01    Agent
Elkeid Agent 的设计还是非常典型的，为笔者丰富了此类主机 Agent 开发视角。Elkeid Agent 提供端上组件的基本能力支撑，包括数据通信（RPC stream）、资源监控、组件版本控制、文件传输、机器基础信息采集等。Agent 本身作为一个插件底座以系统服务的方式运行，其他子插件的策略存放于服务器端的配置，Agent 接收到相应的控制指令及配置后对自身及插件进行开启、关闭、升级等操作

Agent 与 AgentServer 之间采用 bi-stream gRPC（TLS） 进行 [通信](https://github.com/pandaychen/elkeid_fork/blob/main/server/agent_center/grpctrans/proto/grpc.proto#L102)，一些细节如下：

1.  Agent -> AgentServer 方向为数据通道（上报），子 plugin 获取的数据会通过此链路上报
2.  AgentServer -> Agent 方向为指令通道（控制流），作为通知 Agent 执行相关的指令等，见 [此](https://github.com/pandaychen/elkeid_fork/blob/main/server/agent_center/grpctrans/proto/grpc.proto#L31C9-L31C19)，使用 protobuf 的不同 message 类型来标识不同指令类型
3.  Agent 本身支持客户端侧服务发现，也支持跨 Region 级别的通信配置，实现一个 Agent 包在多个网络隔离的环境下进行安装，基于底层一个 TCP 连接，在上层实现了 `Transfer` 与 `FileOp` 两种数据传输服务，支撑了插件本身的数据上报与 Host 中的文件交互
4.  Plugins（安全能力插件）与 Agent 为 **父/子** 进程关系，以两个 `os.Pipe()` 作为跨进程通信方式，Plugins 实现了 Go/Rust 的两个插件库，负责插件端信息的编码与发送（参考下图）。Plugins 发送数据后，会被编码为 Protobuf 二进制数据，Agent 接收到后无需二次解编码，在其外层拼接好 `Header` 特征数据，直接传输给 Server，一般情况下 Server 也无需解码，直接传输至后续数据流中，使用时进行解码，一定程度上降低了数据传输中多次编解码造成的额外性能开销，这里的逻辑参考 `WriteEncodedRecord` 的 [实现](https://github.com/pandaychen/elkeid_fork/blob/main/agent/plugin/plugin_linux.go#L158)，这里的实现思路非常值得借鉴
5.  Agent 采用 Go 开发，在 Linux 下，通过 `systemd` 作为守护方式，通过 `cgroup` 限制资源使用

![pipe]()

##  0x02 Agent核心代码
Agent主要包含三个子模块

-   heartbeat：处理自身host层面/各个插件的核心运行指标上报
-   plugin：处理各个子插件的启动/停止（），负责与各个子插件的上行（指令），下行（数据）通信
-   transport：负责与AgentServer通信，作为stream RPC的客户端，负责打通plugin模块与AgentServer的上下行传输通道

####    heartbeat

####    plugin

####    transport

##  0x03    AgentServer

####    数据流

####    指令流


##  0x04    参考
-   [最后防线：三款开源 HIDS 功能对比评估](https://mp.weixin.qq.com/s?__biz=MzU4NjY0NTExNA==&mid=2247488176&idx=1&sn=879142f6acbdcb291b432f8d4f6a45aa&chksm=fdf979a5ca8ef0b3b979a8cfe9b27dc8885427eab347fb31c94c2e43c6eac72eae8066652fe7&scene=178&cur_album_id=1783332321308229634#rd)
-   [如何利用 Elkeid 发现生产网内恶意行为](https://www.anquanke.com/post/id/250881)
-   [手把手带你部署 Elkeid 开源项目](https://zhuanlan.zhihu.com/p/498091918)