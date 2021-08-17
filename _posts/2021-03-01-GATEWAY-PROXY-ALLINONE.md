---
layout: post
title: 项目开发：网关（Gateway）与反向代理（ReverseProxy）的那些事
subtitle:
date: 2021-03-01
author: pandaychen
header-img:
catalog: true
category: false
tags:
  - Gateway
  - 网关
  - 反向代理
  - ReverseProxy
---

## 0x00 前言

本文是网关及代理开发过程中的一些总结。代理（常见正向代理及反向代理）可以视为网关的核心组件之一。常见的网关有：

- 安全网关（Secure Gateway）
- 开放认证网关（Open IDP Gateway）
- 微服务网关（MicroService Gateway）

对网关的特性主要关注下面几点：

1. 功能 && 部署架构
2. 高性能
3. 高可用
4. 插件式应用，如中间件等
5. 合理集成微服务的特性，如 Tracing/Metrics 观测 / 超时传递 / 熔断、限流机制 / 服务发现等等

## 0x01 一种网关的实现

下面是本人近期项目中实现的一个身份认证网关的基础流程：
![a-basic-idp-gateway-workflow](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/gateway/a-basic-idp-gateway-workflow.png)

整个网关的架构大致如下图所示：
![api-gateway](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/gateway/bifrost-api-gateway.png)

## 0x02 实现的细节

#### 使用 http.ReverseProxy 实现反向代理功能

## 0x03 参考

- [Proxy route to another backend](https://github.com/gin-gonic/gin/issues/686)
- [A simple reverse proxy in Go using Gin](https://le-gall.bzh/post/go/a-reverse-proxy-in-go-using-gin/)

转载请注明出处，本文采用 [CC4.0](http://creativecommons.org/licenses/by-nc-nd/4.0/) 协议授权
