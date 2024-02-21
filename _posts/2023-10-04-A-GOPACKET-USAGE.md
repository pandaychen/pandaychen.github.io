---
layout:     post
title:      DPI with go：构建一个流量分析系统
subtitle:
date:       2023-10-04
author:     pandaychen
catalog:    true
tags:
    - DPI
    - gopacket
---


##  0x00    前言
项目中需要实现一些基本的流量分析功能，借用了 gopacket 库，该库是 libpcap 和 npcap 的 go 封装，提供了更方便的 go 语言操作接口。通常网络抓包有以下几个步骤：
1.  枚举主机上网络设备的接口
2.  针对某一网口进行抓包
3.  解析数据包的 mac 层、ip 层、tcp/udp 层字段等
4.  ip 分片重组，或 tcp 分段重组成上层协议如 http 协议的数据
5.  对上层协议进行头部解析和负载部分解析


####    应用场景
-   网络流量分析：对网络设备流量进行实时采集以及数据包分析
-   伪造数据包发送
-   离线 pcap 文件的读取和写入

##  TCP 流重组
TCP 流重组的代码在 [此](https://github.com/google/gopacket/blob/master/reassembly/tcpassembly.go)


##  参考
-   [Provides packet processing capabilities for Go](https://github.com/google/gopacket)
-   [[译] 利用 gopackage 进行包的捕获、注入和分析](https://colobu.com/2019/06/01/packet-capture-injection-and-analysis-gopacket/)
-   [tcp 重组](https://github.com/google/gopacket/blob/master/reassembly/tcpassembly.go)
-   [tcp 重组 - 实现（推荐）](https://github.com/google/gopacket/blob/master/reassembly/)