---
layout:     post
title:      GOST 应用与分析
subtitle:
date:       2024-01-21
author:     pandaychen
catalog:    true
tags:
    - Gost
---


##  0x00    前言
gost 是一个非常有意思的项目，在笔者看来，像是积木一样的代理连接器，其核心概念是四大模块：

-   `Service`：Service 是指一个网络服务，它可以是一个服务器或者一个客户端。每一个 service 都有一个特定的网络地址和网络协议，如 HTTP，SOCKS5 等。GOST 通过 service 来接收和发送网络数据
-   `Node`：Node 代表一个代理服务器。一个 node 包含了代理服务器的地址、端口和协议等信息。一个 Service 可以由一个或多个 node 组成
-   `Hop`：Hop 是指在 GOST 中数据传输过程中，从一个 node 到另一个 node 的跳转。每个 hop 都有一个特定的转发规则，例如从一个 HTTP 代理跳转到一个 SOCKS5 代理
-   `Chain`：Chain 是指一个网络服务链路。一个 chain 可以包含多个 hop，数据在这些 hop 之间按照特定的顺序进行转发。通过 chain，GOST 可以实现数据的多次转发，从而实现复杂的网络代理功能


####    支持 tunnel 场景
GOST 作为隧道有三种主要使用方式：

####    正向代理
作为代理服务访问网络，可以组合使用多种协议组成转发链进行转发
![forwarder](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/gost/v3-forwarder.png)

####    端口转发
将一个服务的端口映射到另外一个服务的端口，同样可以组合使用多种协议组成转发链进行转发
![portforwarding](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/gost/v3-port-flow.png)

####    反向代理
利用隧道和内网穿透将内网服务暴露到公网访问
![reverse](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/gost/v3-reverse.png)


先看一个简单的 [例子](https://gost.run/getting-started/quick-start/)：

####    代理模式
1、代理 + 转发：监听在 `8080` 端口的 HTTP 代理服务，使用 `192.168.1.1:8080` 做为上级代理进行转发

```YAML
services:
- name: service-0
  addr: ":8080"
  handler:
    type: http
    chain: chain-0
  listener:
    type: tcp
chains:
- name: chain-0
  hops:
  - name: hop-0
    nodes:
    - name: node-0
      addr: 192.168.1.1:8080
      connector:
        type: http
      dialer:
        type: tcp
```

2、使用多级转发（转发链）：GOST 按照设置 `hop-0`、`hop-1` 的顺序将请求最终转发给 `192.168.1.2:1080` 处理

```yaml
services:
- name: service-0
  addr: ":8080"
  handler:
    type: auto
    chain: chain-0
  listener:
    type: tcp
chains:
- name: chain-0
  hops:
  - name: hop-0
    nodes:
    - name: node-0
      addr: 192.168.1.1:8080
      connector:
        type: http
      dialer:
        type: tcp
  - name: hop-1
    nodes:
    - name: node-0
      addr: 192.168.1.2:1080
      connector:
        type: socks5
      dialer:
        type: tcp
```

####    转发模式
1、TCP 本地端口转发：将本地的 TCP 端口 `8080` 映射到 `192.168.1.1` 的 `80` 端口，即所有到本地 `8080` 端口的数据会被转发到 `192.168.1.1:80`

```YAML
services:
- name: service-0
  addr: :8080
  handler:
    type: tcp
  listener:
    type: tcp
  forwarder:
    nodes:
    - name: target-0
      addr: 192.168.1.1:80
```

2、UDP 本地端口转发：将本地的 UDP 端口 `10053` 映射到 `192.168.1.1` 的 `53` 端口，所有到本地 `10053` 端口的数据会被转发到 `192.168.1.1:53`

```YAML
services:
- name: service-0
  addr: :10053
  handler:
    type: udp
  listener:
    type: udp
  forwarder:
    nodes:
    - name: target-0
      addr: 192.168.1.1:53
```

3、TCP 本地端口转发（转发链）：将本地的 TCP 端口 `8080` 通过转发链 `chain-0` 映射到 `192.168.1.1` 的 `80` 端口

```YAML
services:
- name: service-0
  addr: :8080
  handler:
    type: tcp
    chain: chain-0
  listener:
    type: tcp
  forwarder:
    nodes:
    - name: target-0
      addr: 192.168.1.1:80
chains:
- name: chain-0
  hops:
  - name: hop-0
    nodes:
    - name: node-0
      addr: 192.168.1.2:1080
      connector:
        type: socks5
      dialer:
        type: tcp
```

4、TCP 远程端口转发：在 `192.168.1.2` 上开启并监听 TCP 端口 `2222`，并将 `192.168.1.2` 上的 `2222` 端口映射到本地 TCP 端口 `22`，所有到 `192.168.1.2:2222` 的数据会被转发到本地端口 `22`

```YAML
services:
- name: service-0
  addr: :2222
  handler:
    type: rtcp
  listener:
    type: rtcp
    chain: chain-0
  forwarder:
    nodes:
    - name: target-0
      addr: :22
chains:
- name: chain-0
  hops:
  - name: hop-0
    nodes:
    - name: node-0
      addr: 192.168.1.2:1080
      connector:
        type: socks5
      dialer:
        type: tcp
```

5、UDP 远程端口转发：在 `192.168.1.2` 上开启并监听 UDP 端口 `10053`，并将 `192.168.1.2` 上的 `10053` 端口映射到本地 UDP 端口 `53`，所有到 `192.168.1.2:10053` 的数据会被转发到本地端口 `53`

```YAML
services:
- name: service-0
  addr: :10053
  handler:
    type: rudp
  listener:
    type: rudp
    chain: chain-0
  forwarder:
    nodes:
    - name: target-0
      addr: :53
chains:
- name: chain-0
  hops:
  - name: hop-0
    nodes:
    - name: node-0
      addr: 192.168.1.2:1080
      connector:
        type: socks5
      dialer:
        type: tcp
```


##  0x01    tunnel 实现
本小节分析下 [gost 项目](https://github.com/go-gost/gost?tab=readme-ov-file) 的 tunnel 实现，所谓 tunnel，就是通过此传输一些其他协议的数据（变换协议）或者加速访问（和 frp 功能类似）

本文分析版本基于 [v3.0.0-nightly.20240201](https://github.com/go-gost/gost/releases/tag/v3.0.0-nightly.20240201)


##  0x0 参考
-   [GOSTV3](https://github.com/go-gost/gost)
-   [GOSTV2](https://github.com/ginuerzh/gost)
-   [GOST 官方文档](https://gost.run/en/)
-   [GOST - 概述](https://gost.run/concepts/architecture/)