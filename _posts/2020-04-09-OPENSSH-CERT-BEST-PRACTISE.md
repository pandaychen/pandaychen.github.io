---
layout:     post
title:      OpenSSH Certificate 最佳实践
subtitle:
date:       2020-04-09
author:     pandaychen
header-img: img/golang-horse-fly.png
catalog: true
category:   false
tags:
    - OpenSSH
---


##  0x00    前言
针对零信任安全领域的 SSH 服务器治理，OpenSSH 的证书体系绝对是最佳的安全选择。上一篇文章 [证书（Certificate）的那些事](https://pandaychen.github.io/2019/07/24/auth/) 简单介绍了 OpenSSH 的证书体系。本文针对 OpenSSH Cert 的应用做一个系统的梳理。大概设计如下几个方面，这里 `Cert` 代指 OpenSSH 证书：
-       `Cert` VS 公钥
-       `Cert` 的优化及改造实践
-       `Cert` 和 SSO 的结合（零信任方案）
-       `Cert` 的安全性及不足
-       `Cert` 的其他知识点

##  0x01     `Cert` VS 公钥
首先，这两者是不相同的，但是其实在 OpenSSH 的处理逻辑中，是在相同的流程中处理的逻辑。OpenSSH `Cert` 也是基于密钥认证的, `Cert` 是经过 CA 签名后的公钥（回顾上一篇文章的内容）：<br>

$$ 证书 = 公钥 + 元数据 (公钥指纹 / 签发 CA / 序列号 / 有效日期 / 登录用户等)$$

####    算法安全

常用的 SSH 登录秘钥生成算法有如下四种：
-   `DSA`
-   `RSA`
-   `ECDSA`
-   `ED25519`

在安全性上，`DSA` 和 `RSA` 是易于对两个极大质数乘积做质因数分解的困难度，而 `ECDSA`, `ED25519` 则是基于椭圆曲线的离散对数难题。

总结来说：这 4 种算法的推荐排序如下（推荐使用 `ED25519` 算法）：<br>
🚨 DSA: It’s unsafe and even no longer supported since OpenSSH version 7, you need to upgrade it!

⚠️ RSA: It depends on key size. If it has 3072 or 4096-bit length, then you’re good. Less than that, you probably want to upgrade it. The 1024-bit length is even considered unsafe.

👀 ECDSA: It depends on how well your machine can generate a random number that will be used to create a signature. There’s also a trustworthiness concern on the NIST curves that being used by ECDSA.

✅ Ed25519: It’s the most recommended public-key algorithm available today!

##      0x02    `Cert` 的优化及改造实践
基于 OpenSSH 证书签发 CA, 与我们所熟知的 HTTPS 证书的签发使用的 `X.509` 体系不同, 它不支持 证书链（Cert Chain） 和 可信商业 CA。在项目实践中，我们基于 OpenSSH 证书做了大量的安全性提升的工作。

####    用户认证
基于 CA 签发的用户证书主要用于 SSH 登录，如下面一个证书，我们可以基于 `key ID` 或者 `Critical Options` 这个字段做些额外的工作。此外，作为登录使用的证书对，建议满足如下几点（提升证书使用的安全性）：
1.      证书签发的生效时间区间尽量缩短（快速过期）
2.      证书的登录用户唯一（最小化签发）
3.      一次一签 VS 一机一证书
```javascript
 Type: ssh-ed25519-cert-v01@openssh.com user certificate
        Public key: ED25519-CERT SHA256:wdzTWhCrVeJrxRIC1KU5nJr8FbxxCUJt1IVeG7HYjmc
        Signing CA: ED25519 SHA256:OEhTm77qM7ZDwb5oltxt78FIpKraXCzxoaboi/KpNbM
        Key ID: "08a093ec-cb4e-4bc2-9800-825095418397:981b88e2-a214-4075-af77-72da9600f34f"
        Serial: 0
        Valid: from 2019-07-31T11:21:00 to 2019-07-31T12:22:50
        Principals:
                root
                pandaychen
        Critical Options: (none)
        Extensions:
                permit-X11-forwarding
                permit-agent-forwarding
                permit-port-forwarding
                permit-pty
                permit-user-rc
```

####    主机认证
主机证书主要用于替换服务器的 Hostkey 认证，用于服务端告诉客户端，我是经由 CA 签发（认证）的合法服务器：
```javascript
ssh_host_ecdsa_key-cert.pub:
        Type: ecdsa-sha2-nistp256-cert-v01@openssh.com host certificate
        Public key: ECDSA-CERT 51:7e:99:5d:dc:05:9e:21:85:d1:e1:10:d3:a3:77:8a
        Signing CA: RSA d9:a2:2f:ca:f5:15:9b:9e:0b:c6:5e:4e:bb:4d:3e:fd
        Key ID: "08a093ec-cb4e-4bc2-9800-825095418397:981b88e2-a214-4075-af77-72da9600f123e"
        Serial: 0
        Valid: forever
        Principals: (none)
        Critical Options: (none)
        Extensions:
                permit-X11-forwarding
                permit-agent-forwarding
                permit-port-forwarding
                permit-pty
                permit-user-rc
```

####    其他安全特性
由于证书的不可伪造性（Unforgeability），我们可以利用证书的内置字段或结构来提升证书使用的安全性。此外，OpenSSH 还支持多个 CA （公钥）共用（虽然不推荐这样配置）

##      0x03    `Cert` 和 SSO 的结合（零信任方案）
CloudFlare 的 OpenSSH 实践：[Public keys are not enough for SSH security](https://blog.cloudflare.com/public-keys-are-not-enough-for-ssh-security/)，文中给出了一个非常值得借鉴的 OpenSSH 证书架构与 SSO 结合的安全登录体系。整体架构图如下：

![img](https://blog-cloudflare-com-assets.storage.googleapis.com/2019/10/Short-lived-Cert@2x.png)

图中的流程大致如下：
1.      用户发起 SSH 登录请求
2.      Cloudflare Access 的认证流程，这里可以采用 Oauth、OneLogin 等等开放的认证体系来完成，另外推荐也加入 2FA
3.      Cloudflare Access 为用户生成 JWT 票据
4.      Cloudflare CA 验证 JWT 票据，验证 ok 后请求签发新的客户端 SSH 登录证书
5.      Cloudflare CA 完成签发证书并返回
6.      日志审计
7.      此时用户可以使用 JWT 票据 +`short-lived certificates` 登录服务器（当然管理员需要实现在目标服务器上部署证书的公钥）


##  0x04    `Cert` 的安全性及不足

####    CA 密钥安全
虽然 OpenSSH 证书登录的方案，集中化的 CA 密钥管理简化了认证管理流程，不幸的是这同时也简化了攻击面，攻击者只许获得 CA 密钥的管理就能攻破相应机器的 SSH 登录。所以，对 CA 的保护是非常核心的一环：
1.      对 CA 的加密存储，可以使用 `KMS` 或者使用复杂的加密算法加密后存储
2.      CA 私钥的定期轮换机制

##      0x05    `Cert` 的其他知识点


##  0x06    参考
-       [How Uber, Facebook, and Netflix Do SSH](https://gravitational.com/blog/how_uber_netflix_facebook_do_ssh/)
-   [How to SSH Properly](https://gravitational.com/blog/how-to-ssh-properly/)
-   [Public keys are not enough for SSH security](https://blog.cloudflare.com/public-keys-are-not-enough-for-ssh-security/)
-   [If you’re not using SSH certificates you’re doing SSH wrong](https://smallstep.com/blog/use-ssh-certificates/)
-   [Upgrade Your SSH Key to Ed25519](https://medium.com/risan/upgrade-your-ssh-key-to-ed25519-c6e8d60d3c54)
-   [使用 openssh 证书认证](https://wooyun.js.org/drops/%E4%BD%BF%E7%94%A8OpenSSH%E8%AF%81%E4%B9%A6%E8%AE%A4%E8%AF%81.html)
-   [Blessing your SSH at Lyft](https://eng.lyft.com/blessing-your-ssh-at-lyft-a1b38f81629d)
-   [Scalable and secure access with SSH](https://engineering.fb.com/security/scalable-and-secure-access-with-ssh/)
-   [Introducing the Uber SSH Certificate Authority](https://medium.com/uber-security-privacy/introducing-the-uber-ssh-certificate-authority-4f840839c5cc)
-   [Netflix-bless](https://github.com/Netflix/bless)
-   [HashiCorp Vault SSH CA and Sentinel](https://medium.com/hashicorp-engineering/hashicorp-vault-ssh-ca-and-sentinel-79ea6a6960e5)
-   [Signed SSH Certificates](https://www.vaultproject.io/docs/secrets/ssh/signed-ssh-certificates.html#known-issues)

转载请注明出处，本文采用 [CC4.0](http://creativecommons.org/licenses/by-nc-nd/4.0/) 协议授权