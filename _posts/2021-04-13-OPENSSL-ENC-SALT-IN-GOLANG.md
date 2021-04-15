---
layout: post
title: 在 Golang 中实现 Openssl 的 AES-CBC-256 算法（With Salt）--未完成
subtitle:
date: 2021-04-15
author: pandaychen
header-img:
catalog: true
category: false
tags:
  - Openssl
  - Golang
  - AES
---

## 0x00 前言

今天在使用 `openssl` 工具中遇到如下的 case：

```bash
plaintext="I'm password"
password="abcdefghijklmn"
echo $plaintext | openssl enc -salt -aes-256-cbc -e -a -k $password
encrypted=`echo $plaintext | openssl enc -salt -aes-256-cbc -e -a -k $password`
echo $encrypted | openssl enc -salt -aes-256-cbc -d -a -k $password
```

其中某一次的加密结果是 `U2FsdGVkX1+m6PD2vAhrOI8F5HhC0NlctfX0DvuHOYI=`，仔

一看是常见的 `aes-256-cbc` 加密模式，其中某一次的加密结果是 `U2FsdGVkX1+m6PD2vAhrOI8F5HhC0NlctfX0DvuHOYI=`，这与平时我们使用的 `aes-256-cbc` 有些出入：

- 加密串长度不对
- 加密的结果每次都不同

研究了下，原来是 `-salt` 这个参数在搞鬼。官方文档对此选项的解释是：salt 是一个随机数，salt 与 passwd 串联，然后计算其 hash 值来防御 dictionary attacks 和预计算的 rainbow table 攻击。在 `openssl` 的 `enc` 命令中，通过 salt 与 passwd 来生加密（解密）密钥和初始向量 IV。

下面分析下加了 `- salt` 选项之后的加解密过程及其 golang 实现。

## 0x01 加密原理

## 0x02 Golang 实现

## 0x03 参考

- [openssl 学习之 enc 中 salt 参数解析](https://blog.csdn.net/kkxgx/article/details/12879367)
- [Yet Another Padding Oracle in OpenSSL CBC Ciphersuites](https://blog.cloudflare.com/yet-another-padding-oracle-in-openssl-cbc-ciphersuites/)

转载请注明出处，本文采用 [CC4.0](http://creativecommons.org/licenses/by-nc-nd/4.0/) 协议授权
