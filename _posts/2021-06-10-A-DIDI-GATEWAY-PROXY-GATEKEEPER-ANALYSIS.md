---
layout: post
title: DiDi 开源 API 网关 gatekeeper 项目分析
subtitle: 一个微服务网关的设计与实现
date: 2021-06-10
author: pandaychen
header-img:
catalog: true
category: false
tags:
  - Gateway
  - 微服务网关
---

## 0x00 前言

[Didi-gatekeeper](https://github.com/didi/gatekeeper) 是一个 `Golang` 的不依赖分布式数据库的 `API` 网关（基于 `gin`），使用它可以高效进行服务代理，支持在线化热更新服务配置以及纯文件方式服务配置，支持主动探测方式自动剔除故障节点以及手动方式关闭下游节点流量，还可以通过自定义中间件方式灵活拓展其他功能。

本文主要分析下其实现中可以借鉴的思路及细节：

- 插件式中间件引入
- 部署架构

其 [官网文档](https://github.com/didi/gatekeeper/blob/master/README.md) 有比较详细的项目介绍，特性为：

- 支持 `http`、`websocket`、`tcp` 服务代理
- 自动剔除故障节点
- 手动关闭下游节点流量
- 加权负载轮询
- `URL` 地址重写
- 服务限流：支持独立 `IP` 限流
- 高拓展性：如支持自定义请求前验证 `request`、请求后更改 `response`、tcp 中间件、http 中间件等方法
- 最少依赖：无需任何额外组件即可运行，`mysql`、`redis` 只做在线管理和统计使用，可随时关闭

## 0x01 部署架构图

![img](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/didi-gatekeeper/architecture1.png)

`GateKeeper` 的架构图是比较典型的网关层次架构，前端使用通用的 Nginx 接入，`GateKeeper` 本身可以做无状态的平行扩展

- 接入层（`Nginx`，`Haproxy` 等）
- 网关层（包含配置中心、管理平面、健康度监控等）
- 业务层

流程如下：

1. 用户通过接入层连接到 `GateKeeper` 实例，接入层一般选用 `nginx`、`Haproxy`、`LVS` 等高可用 HA 集群（当然也可以直接让 `GateKeeper` 实例对外，不推荐）
2. 每个 `GateKeeper` 实例，针对每个服务模块，单独进行服务探测
3. 在线服务管理时，配置数据先保存到 `GateKeeper` 配置 `DB` 中，然后再通过调用配置更新接口（ `/reload` ），更新所有实例机器配置

## 0x02

## 0x03 参考

- [gatekeeper repo](https://github.com/didi/gatekeeper)

转载请注明出处，本文采用 [CC4.0](http://creativecommons.org/licenses/by-nc-nd/4.0/) 协议授权
