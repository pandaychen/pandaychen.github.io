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

####  WAF (Web应用防火墙)
防止CC攻击
防SQL注入、XSS等
防敏感数据泄露
HTTPS支持，不需要Agent

##  0x03  
统一的Web化管理
统一的Web化管理
Web SSH安全运维
OAuth2身份认证


数字证书保护
私钥加密存储
不使用文件形态的证书
内存解密使用

可扩展的架构
多网关/负载均衡
私有化部署
自动策略同步
软件形式交付

## 0x0 参考
-	[数据安全架构与治理](https://doc.janusec.com/download/Janusec-Application-Gateway.pdf)
-	[零信任下的应用安全网关该如何建设？](https://blog.csdn.net/a59a59/article/details/103763203)
- [Zero trust system](https://github.com/pritunl/pritunl-zero)
- [应用管理](https://doc.janusec.com/cn/application-management/)
- [阿里云-安全办公平台](https://help.aliyun.com/product/181210.htm?spm=5176.19595751.J_3643955890.5.5cf254a3yqNhOg)
- [《零信任网络》系列连载 (一)：一文读懂零信任网络](https://www.secrss.com/articles/12798)
- [中通零信任安全代理ZFE](https://www.secrss.com/articles/29068)
- [零触碰与零信任](https://res-www.zte.com.cn/mediares/magazine/publication/com_cn/article/202103/10.pdf)

转载请注明出处，本文采用 [CC4.0](http://creativecommons.org/licenses/by-nc-nd/4.0/) 协议授权
