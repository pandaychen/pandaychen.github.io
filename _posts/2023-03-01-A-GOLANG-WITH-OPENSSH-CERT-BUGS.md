---
layout:     post
title:      OpenSSH Certificate 与 Golang 的兼容性问题
subtitle:
date:       2023-03-01
author:     pandaychen
header-img: img/golang-horse-fly.png
catalog: true
category:   false
tags:
    - OpenSSH
    - Golang
---


##  0x00    前言
前文 [OpenSSH Certificate 证书最佳实践](https://pandaychen.github.io/2020/04/09/OPENSSH-CERT-BEST-PRACTISE/) 介绍了证书的实践，不过近期笔者在工作中遇到了非常麻烦的兼容性问题，这里摘录几个：

-   [x/crypto/ssh: "ssh-rsa-cert-v01@openssh.com" does not work for sshd OpenSSH 7.2-7.7 #58371](https://github.com/golang/go/issues/58371)
-   [x/crypto/ssh: cannot sign certificate with different algorithm #36261](https://github.com/golang/go/issues/36261)
-   [x/crypto/ssh: rsa-sha2-256/rsa-sha2-512 tracking issue](https://github.com/golang/go/issues/49952#issuecomment-1314152075)

以上的问题核心点是兼容性问题，主要涉及到三方的兼容性问题：
1.  golang（`golang.org/x/crypto/ssh`）版本（也有可能是二进制版本），主要影响两块：
    -   CA 服务端：证书签发（不同 go 版本编译的生成证书的格式可能不一样，如 `1.11` 和 `1.17` 编译出的 CA 服务生成的 `RSA-2048` 证书）
    -   SSH 客户端：证书解析及登录（主要是解析证书，`1.11` 版本的客户端不一定能解析出 `1.17` 版本生成的证书）
2.  Openssh：sshd 版本（服务器），对算法、证书类型有兼容性要求
3.  Openssh：ssh 版本（客户端），包括 `ssh-keygen` 工具的兼容性问题

##      0x01    兼容性测试
现网使用的 go 版本主要是：`go1.11` 以及 `go.17`


##      0x02    原因


##  0x03    总结

##  0x04    参考
-   [Use RSA CA Certificates with OpenSSH 8.2](https://ibug.io/blog/2020/04/ssh-8.2-rsa-ca/)
-   [SSH Certificates: How Do OpenSSH Certificates Compare to X.509?](https://goteleport.com/blog/x509-vs-openssh-certificates/)

转载请注明出处，本文采用 [CC4.0](http://creativecommons.org/licenses/by-nc-nd/4.0/) 协议授权
