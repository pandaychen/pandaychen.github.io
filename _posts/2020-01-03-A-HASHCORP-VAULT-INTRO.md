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
我们在工作中是如何管理大量的 Secret 信息的？
-   以配置文件的形式固化，存放于服务器文件或者 Database 中
-   以代码的方式存在于 `git` 私有仓库上，并严格控制此库的访问权限
-   以 KMS（Key Management Service，云服务居多）方式托管在公有云服务上

本文要介绍 [Vault](https://learn.hashicorp.com/vault) 这款开源的 Secret 管理工具（口令、token、私钥及证书等等），是管理代码中口令、秘钥（防止明文泄漏）极佳的应用实践。此外，KMS（Key Management Service，云服务居多）也是较好的 Secret 管理实践。源码 [在此](https://github.com/hashicorp/vault)

##  0x01    Vault 基本原理

####    Shamir 密钥分享算法（shamir secret sharing）
Vault 中给出了 Shamir 算法的 [实现](https://github.com/hashicorp/vault/blob/master/shamir/shamir.go)

####    存储
Vault 支持多种存储后端：具体 [在此](https://github.com/hashicorp/vault/tree/master/plugins/database)

-   [mysql](https://github.com/hashicorp/vault/blob/master/plugins/database/mysql/mysql.go)


##  参考
-   [vault - Documentation](https://www.vaultproject.io/docs)
-   [Secret sharing](https://en.wikipedia.org/wiki/Secret_sharing)
-   [密钥分享 Secret Sharing 介绍](https://zhuanlan.zhihu.com/p/44999983)
-   [小马哥 Devops](https://my.oschina.net/u/3952901?tab=newest&catalogId=7038751)
