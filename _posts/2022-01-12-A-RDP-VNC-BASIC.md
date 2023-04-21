---
layout: post
title: 使用 Golang 实现 RDP 和 VNC（一）
subtitle: rdp/vnc 协议基础 && 使用（入门篇）
date: 2022-01-12
author: pandaychen
catalog: true
tags:
  - Golang
  - RDP
  - VNC
---

## 0x00 前言
（RDP）Windows Remote Desktop Protocol、VNC（Virtual Network Computing）都是图形化的登录终端（远程桌面）协议。本文介绍基础概念及使用，涉及如下知识，参考 [此文](https://mojotv.cn/golang/golang-html5-websocket-remote-desktop)：
- RDP：RDP 远程桌面协议（支持 linux/windows）
- VNC：linux 中常用的屏幕分享协议
- Guacamole Protocol：(guacamole.apache.org 中的协议)

远程桌面协议是一个多通道（multi-channel）的协议，让用户（本地电脑）连上提供微软终端服务的电脑（远程电脑）。大部分的 Windows 都有客户端软件。其他操作系统例如 Linux、FreeBSD、Mac OS X，也有对应的客户端软件

VNC 为一种使用 RFB 协议的屏幕画面分享及远程操作软件。此软件借由网络，可发送键盘与鼠标的动作及即时的屏幕画面。 **VNC 与操作系统无关 **，因此可跨平台使用，例如可用 Windows 连线到某 Linux 的电脑，反之亦同。甚至在没有安装客户端程序的电脑中，只要有支持 JAVA 的浏览器，也可使用

Guacamole 是 Apache 出品的免费开源远程桌面网关，**通过 Guacamole，无需任何客户端或插件，只要有支持 HTML5 和 JavaScript 的 Web 浏览器即可访问远程资源，不仅支持 Windows RDP 协议，也支持 VNC 协议，甚至还支持 SSH、Telnet 等协议。**Guacamole 的核心目标是将桌面保持在云端，从任何地方访问计算机。


##  0x01  RDP 基础


##  0x02  VNC 基础

通过 VNC（Virtual Network Computing）和远程桌面来连接到远程的计算机上，二者区别如下：
- VNC：VNC 使用的是 RFB 协议来做屏幕分享和远程操作的软件，由著名的 AT&T 欧洲研究实验室开发。使用时需要客户端软件和服务器软件配合使用
- 远程桌面：远程桌面使用 RDP(Remote Desktop Protocol) 远程桌面协议，大部分 Windows 都默认支持此协议，可以远程接入操作桌面

####  RFB 和 RDP 区别
RFB 是在服务器端将窗口在显存中画好之后将图像传给客户端，客户端只充当一个图像解码显示的角色； RDP 则将画图的工作交给了客户端，服务器需要了解客户端显示能力的具体情况，以便作出相应调整。

RFB 主要传图像，RDP 主要传指令。就一般应用而言，RFB 数据量太大，RDP 对客户端要求较高，因此 RFB 适用于瘦客户端，RDP 适用于低速网络

##  0x03  Guacamole 基础
guacamole 架构图如下：
![]()


##  0x04  rdpgo 分析
[rdpgo](https://github.com/mojocn/rdpgo) 项目实现了 websocket 到 guacamole 协议的转换


## 0x0 参考
-	[Go 进阶 53: 从零 Go 实现 Websocket-H5-RDP/VNC 远程桌面客户端](https://mojotv.cn/golang/golang-html5-websocket-remote-desktop)
- [All in Web | 远程桌面网关 - Apache Guacamole](https://zhuanlan.zhihu.com/p/432814073)
- [Guacamole 源码分析与 VNC 中 RFB 协议的坑](https://changkun.de/blog/posts/guacamole-%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%E4%B8%8E-vnc-%E4%B8%AD-rfb-%E5%8D%8F%E8%AE%AE%E7%9A%84%E5%9D%91/?hmsr=joyk.com&utm_source=joyk.com&utm_medium=referral)
- [Guacamole Protocol 技术文档](https://guacamole.apache.org/doc/gug/guacamole-protocol.html#guacamole-protocol-handshake)
- [rdpgo：Websocket-H5-RDP/VNC 远程桌面客户端](https://github.com/mojocn/rdpgo)
- [vncproxy](https://github.com/vprix/vncproxy)
- [guac：Apache Guacamole client ported to Go](https://github.com/wwt/guac)
- [go-rustdesk-server](https://github.com/danbai225/go-rustdesk-server)