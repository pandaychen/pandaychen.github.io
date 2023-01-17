---
layout: post
title: 一个数据应用网关的实现分析：JANUSEC
subtitle: 基于 golang 的应用网关
date: 2023-01-10
author: pandaychen
header-img:
catalog: true
category: false
tags:
  - Gateway
  - 微服务网关
---

## 0x00 前言
[Janusec](https://github.com/Janusec/Application-Gateway) 应用网关，提供 WAF（Web 应用防火墙）、CC 防护、身份认证、安全运维、Web 路由、负载均衡、自动化证书等功能，可用于构建安全的、可扩展的应用。产品 [文档](https://doc.janusec.com/cn/introduction/)

Janusec 的特点如下：
- 应用网关：统一 HTTPS 接入，统一管理证书和秘钥
- 安全防护：网关内置 Web 应用防火墙、CC 防护
- 数据保护：对私钥采取加密措施
- 负载均衡：前端负载均衡与后端负载均衡
- `4-7` 层审计：审计留痕是应用安全网关应具有的功能，在提供安全防护的同时，记录 `4-7` 层数据信息，方便异常问题时的回溯分析
- 应用访问控制：需要依托高性能的规则引擎机制，主要从应用提取相应规则固化至规则引擎系统中，通过对规则评判实现业务访问控制
- 可开发性：基于 Golang 的开源系统，可实现与用户管理结合。如身份认证、信任评估与应用访问控制结合

##  0x01  应用网关


##  0x02  安全防护


## 0x0 参考
-	[数据安全架构与治理](https://doc.janusec.com/download/Janusec-Application-Gateway.pdf)
-	[零信任下的应用安全网关该如何建设？](https://blog.csdn.net/a59a59/article/details/103763203)

转载请注明出处，本文采用 [CC4.0](http://creativecommons.org/licenses/by-nc-nd/4.0/) 协议授权
