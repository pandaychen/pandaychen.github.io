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

一看是常见的 `aes-256-cbc` 加密模式，其中某一次的加密结果是 `U2FsdGVkX1+m6PD2vAhrOI8F5HhC0NlctfX0DvuHOYI=`，这与平时我们使用的 `aes-256-cbc` 有些出入：

- 加密串长度不符合预期结果（和普通方式相比）
- 加密的结果每次都不相同

研究了下，原来是 `salt` 这个参数在搞鬼。官方文档对此选项的解释是：salt 是一个随机数，salt 与 passwd 串联，然后计算其 hash 值来防御 dictionary attacks 和预计算的 rainbow table 攻击。在 `openssl` 的 `enc` 命令中，通过 salt 与 passwd 来生加密（解密）密钥和初始向量 IV

下面分析下加了 `salt` 选项之后的加解密过程及其 golang 实现。

## 0x01 加密原理

关于`salt`的加密流程如下，摘录自[OpenSSL - Password vs Salt Purpose](https://stackoverflow.com/questions/17297637/openssl-password-vs-salt-purpose/17297740)：
In OpenSSL, the salt will be prepended to the front of the encrypted data, which will allow it to be decrypted. The purpose of the salt is to prevent dictionary attacks, rainbow tables, etc. The following is from the OpenSSL documentation:

> Without the -salt option it is possible to perform efficient dictionary attacks on the password and to attack stream cipher encrypted data. The reason for this is that without > the salt the same password always generates the same encryption key. When the salt is being used the first eight bytes of the encrypted data are reserved for the salt: it is > > generated at random when encrypting a file and read from the encrypted file when it is decrypted.

### 测试

1、Password without SALT<br>

```bash
echo "secret data in my file" > plaintext.txt

openssl enc -aes-128-cbc -nosalt -k "mySecretPassword" -in plaintext.txt -out enc1.nosalt.bin
openssl enc -aes-128-cbc -nosalt -k "mySecretPassword" -in plaintext.txt -out enc2.nosalt.bin
```

两次加密的结果是一样的：

```bash
xxd enc1.nosalt.bin
00000000: 576e a82c 0dac 92d8 5e45 5ef4 3f6f db6a  Wn.,....^E^.?o.j
00000010: 5630 554f 3f28 a0de ae96 91d9 1024 d5ca  V0UO?(.......$..

xxd enc2.nosalt.bin
00000000: 576e a82c 0dac 92d8 5e45 5ef4 3f6f db6a  Wn.,....^E^.?o.j
00000010: 5630 554f 3f28 a0de ae96 91d9 1024 d5ca  V0UO?(.......$..
```

2、Password and SALT<br>

```bash
openssl enc -aes-128-cbc -k "mySecretPassword" -in plaintext.txt -out enc2.salted.bin
openssl enc -aes-128-cbc -k "mySecretPassword" -in plaintext.txt -out enc1.salted.bin
```

两次加密的结果不同：

```bash
xxd enc2.salted.bin
00000000: 5361 6c74 6564 5f5f 9cfe 2d62 a2d4 70b8  Salted__..-b..p.
00000010: aee4 afb5 85c9 76a2 cb04 7e1d 27d9 94d4  ......v...~.'...
00000020: a1b3 c4d6 39b8 f5a8 c300 81b5 b6ed 4cca  ....9.........L.

xxd enc1.salted.bin
00000000: 5361 6c74 6564 5f5f e73c ee5b 701b bba8  Salted__.<.[p...
00000010: fa25 c54e befa 26dc ddb1 3a2d 2bd7 a95b  .%.N..&...:-+..[
00000020: bda9 56f0 4445 f229 3398 4076 1044 dad6  ..V.DE.)3.@v.D..
```

加密的过程如下图：
![aes-cbc-salt](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/crypto/openssl_aes_cbc_salt.png)

## 0x02 Golang 实现

有了上面的加密流程，接下来我们尝试用 golang 实现这种加密方式：

## 0x03 参考

- [OpenSSL - Password vs Salt Purpose](https://stackoverflow.com/questions/17297637/openssl-password-vs-salt-purpose/17297740)
- [openssl 学习之 enc 中 salt 参数解析](https://blog.csdn.net/kkxgx/article/details/12879367)
- [Yet Another Padding Oracle in OpenSSL CBC Ciphersuites](https://blog.cloudflare.com/yet-another-padding-oracle-in-openssl-cbc-ciphersuites/)

转载请注明出处，本文采用 [CC4.0](http://creativecommons.org/licenses/by-nc-nd/4.0/) 协议授权
