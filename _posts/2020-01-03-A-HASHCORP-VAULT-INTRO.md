---
layout:     post
title:      Hashcorp Vault 使用
subtitle:   更安全的 Secret 存储系统：vault
date:       2020-01-03
author:     pandaychen
header-img: img/super-mario.jpg
catalog: true
category:   false
tags:
    - Vault
    - 网络安全
---


##  0x00    前言
我们在工作中是如何管理大量的 Secret 信息的（比如笔者的项目中会涉及到对 OpenSSH 的秘钥及口令存储、定时轮转及外部调用）？
-   以配置文件的形式固化，存放于服务器文件或者 Database 中
-   以代码的方式存在于 `git` 私有仓库上，并严格控制此库的访问权限
-   以 KMS（Key Management Service，云服务居多）方式托管在公有云服务上

本文要介绍 [Vault](https://learn.hashicorp.com/vault) 这款开源的 Secret 管理工具（口令、token、私钥及证书等等），是管理代码中口令、秘钥（防止明文泄漏）极佳的应用实践。此外，KMS（Key Management Service，云服务居多）也是较好的 Secret 管理实践。源码 [在此](https://github.com/hashicorp/vault)

针对此类产品，需要着重关注以下几点：
1.  Secret 的存储方式，支持的存储后端
2.  Secret 的加密方式及算法
3.  系统的权限控制、权限分配（哪些人 / 客户端能访问哪些机器的 Secret）
4.  系统的认证方式，客户端使用什么方式访问 RestFUL-API
5.  系统的 Secret 存储方式及过期机制
6.  系统的高可用性如何保证
7.  系统对外接口的 QPS 及并发性能

##  0x01    Vault 基本原理
vault 的基础应用场景如下：
![vault-basic](https://github.com/pandaychen/pandaychen.github.io/blob/master/blog_img/vault/vault-basic.png)

Vault 的使用场景一般为：
1.  系统用户（如运维同事）通过 `HTTP-Vault-API`、Vault 命令行工具等将 secret data 写入 Vault
2.  Vault 再将加密的数据存储到后端
3.  外部用户（如开发人员，各类脚本或者应用程序）通过 `HTTP-Vault-API`、Vault 命令行工具等方式来获取到仅仅与自己账号相关联的 secret data，这里就涉及到 Valut 的权限细粒度管理


vault 的架构如下：
![vault-architecture](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/vault/vault-%E6%9E%B6%E6%9E%84%E5%9B%BE.png)

####    Shamir 密钥分享算法（shamir secret sharing）
Vault 中给出了 Shamir 算法的 [实现](https://github.com/hashicorp/vault/blob/master/shamir/shamir.go)，该密钥分享算法的基本思想是分发者通过秘密多项式，将秘密 secret 分解为 `n` 个秘密持有者，其中任意至少于 `k` 个秘密均能恢复密文，即某一个秘密通常不能由单个持有者保存，必须将秘密分由多人保管并且只有当多人同时在场时秘密才能得以恢复。

![vault-key-share](https://github.com/pandaychen/pandaychen.github.io/blob/master/blog_img/vault/vault-shamir-key-algorithm.png)

####    存储
Vault 支持多种存储后端：具体 [在此](https://github.com/hashicorp/vault/tree/master/plugins/database)，生产架构的后端存储使用 `HA` 方式进行部署，比如 consul/etcd/mysql 集群等。

-   [mysql](https://github.com/hashicorp/vault/blob/master/plugins/database/mysql/mysql.go)


##  参考
-   [vault - Documentation](https://www.vaultproject.io/docs)
-   [Secret sharing](https://en.wikipedia.org/wiki/Secret_sharing)
-   [密钥分享 Secret Sharing 介绍](https://zhuanlan.zhihu.com/p/44999983)
-   [小马哥 Devops](https://my.oschina.net/u/3952901?tab=newest&catalogId=7038751)
-   [vault-architecture](https://www.vaultproject.io/docs/internals/architecture)