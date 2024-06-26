---
layout:     post
title:      OpenSSH Certificate 证书最佳实践
subtitle:   How Do We USE OpenSSH Certificate Properly?
date:       2020-04-09
author:     pandaychen
header-img: img/golang-horse-fly.png
catalog: true
category:   false
tags:
    - OpenSSH
---


##  0x00    前言
针对零信任安全领域的 SSH 服务器安全管控，OpenSSH 的证书体系绝对是最佳的安全选择。上一篇文章 [证书（Certificate）的那些事](https://pandaychen.github.io/2019/07/24/auth/) 简单介绍了 OpenSSH 的证书体系。本文针对 OpenSSH Certificate 的应用做一个系统的梳理。大概涉及到如下几个方面，本文中 `Certificate` 代指 OpenSSH 证书：
-       `Certificate` VS 公钥
-       `Certificate` 的优化及改造实践
-       `Certificate` 和 SSO 的结合（零信任方案）
-       `Certificate` 的安全性及不足
-       `Certificate` 的其他知识点

##  0x01     Certificate VS 公钥
首先，这两者是不相同的，但是其实在 OpenSSH 的处理逻辑中，是在相同的流程中处理的逻辑。OpenSSH `Certificate` 也是基于密钥认证的, `Certificate` 是经过 CA 签名后的公钥（回顾上一篇文章 [证书（Certificate）的那些事](https://pandaychen.github.io/2019/07/24/auth/) 的内容）：<br>

$$ 证书（Certificate） = 公钥（PublicKey） + 元数据 (公钥指纹 / 签发 CA / 序列号 / 证书有效日期 / 登录用户等)$$

![pubkeyVScert](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/ssh/sshkeyVScert.png)

####    公钥认证
SSH 公钥认证流程如下图所示，SSH 公钥是公开分发的，任何持有它的人都可以使用公钥验证使用其私钥副本签名的消息（signature），SSH 服务器生成一个随机字符串（Challenge），并要求 SSH 客户端对其进行签名。服务器使用 SSH 公钥验证客户端的签名，以证明客户端拥有与可信公钥关联的私钥。

![pub-auth](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/openssh/ssh-certs-public-key-auth.png)

![pub-auth-flow](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/openssh/ssh-certs-public-key-protocol.png)

####    证书认证

####    certificate 的优点

-  Certificates are tied to user identity
-  Certificates automatically expire
-  Certificates can contain SSH restrictions, e.g. forbidding PTY allocation or port forwarding
-  SSH certificates can be synchronized with Kubernetes certificates
-  **Certificates include metadata. This enables role-based access control**（teleport 的这个实现蛮有意思）
-  Certificates solve TOFU (trust on first use) problems. The user and host certificates signed by the same CA establish trust and eliminate the need for TOFU


####    算法安全
常用的 SSH 登录秘钥生成算法有如下四种：
-   `DSA`
-   `RSA`
-   `ECDSA`
-   `ED25519`

在安全性上，`DSA` 和 `RSA` 是易于对两个极大质数乘积做质因数分解的困难度，而 `ECDSA`, `ED25519` 则是基于椭圆曲线的离散对数难题。

总结来说：这 4 种算法的推荐排序如下（推荐使用 `ED25519` 算法，从对低版本的兼容性而言，`RSA` 更为兼容，但建议至少使用 `2048` 位的秘钥）：<br>
<br>
🚨 DSA: It’s unsafe and even no longer supported since OpenSSH version 7, you need to upgrade it!

⚠️ RSA: It depends on key size. If it has 3072 or 4096-bit length, then you’re good. Less than that, you probably want to upgrade it. The 1024-bit length is even considered unsafe.

👀 ECDSA: It depends on how well your machine can generate a random number that will be used to create a signature. There’s also a trustworthiness concern on the NIST curves that being used by ECDSA.

✅ Ed25519: It’s the most recommended public-key algorithm available today!

关于安全性可以参考此文 [Comparing SSH Keys - RSA, DSA, ECDSA, or EdDSA?](https://goteleport.com/blog/comparing-ssh-keys/)

##      0x02    Certificate 的优化及改造实践
基于 OpenSSH 证书签发 CA, 与所熟知的 HTTPS 证书的签发使用的 `X.509` 体系不同, 它不支持证书链（Certificate Chain） 和可信商业 CA。在项目实践中，基于 OpenSSH 证书做了大量的安全性提升的工作。如下，OpenSSH 证书存在两种类型，用户证书（User Certificate）和主机证书（Host Certificate）：

####    用户认证
基于 CA 签发的用户证书主要用于 SSH 登录，如下面这个用户证书，可以基于 `key ID` 或者 `Critical Options` 这个字段做些额外的工作。

```text
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

此外，作为登录使用的证书对，建议满足如下几点（提升证书使用的安全性）：
1.      证书签发的生效时间区间尽量缩短（快速过期）
2.      证书的登录用户唯一（最小化签发）
3.      一次一签
4.      一机一证书

####    主机认证
主机证书主要用于替换服务器的 Hostkey 认证，用于服务端告诉客户端，我是经由 CA 签发（认证）的合法服务器：
```text
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

####    Comparing X.509 properties with OpenSSH certificate
OpenSSH 证书与 `X.509` 是两种不同的证书体系，二者的区别如下图：

![diff](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/openssh/sshcert-vs-x509.png)



####    其他安全特性
由于证书的不可伪造性（Unforgeability），可以利用证书的内置字段或结构来提升证书使用的安全性。此外，OpenSSH 还支持多个 CA （公钥）共用（虽然不推荐这样配置）

##      0x03   OpenSSH Certificate With SSO（零信任方案）
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


##  0x04    Certificate 的安全性及不足

####    CA 密钥安全
虽然 OpenSSH 证书登录的方案，集中化的 CA 密钥管理简化了认证管理流程，不幸的是这同时也简化了攻击面，攻击者只许获得 CA 密钥的管理就能攻破相应机器的 SSH 登录。所以，对 CA 的保护是非常核心的一环：
1.      对 CA 的加密存储，可以使用 `KMS` 或者使用复杂的加密算法加密后存储
2.      CA 私钥的定期轮换机制

####    CA 证书安全
对 CA 证书的管理，在项目应用中主要还是加强对用户证书（User Certificate）的管理，一般按照下面的维度来实施：
1.      证书签发的生效时间区间尽量缩短（快速过期）
2.      证书的登录用户唯一（最小化签发）
3.      一次一签
4.      一机一证书
5.      证书尽可能保存在内存中，如必须落盘在本地，建议登录后立即删除

##      0x05    Certificate 的其他知识点


####    Trust On First Use
首次建立 SSH 连接时，SSH 服务器会发送其公钥以向用户标识自己的身份。用户可以接受 SSH 服务器提供的公钥，并且如果用户第一次连接到该主机，则认​​为该主机是可信的，该方案也称为 Trust On First Use 问题（与此对应的是 man in the middle 问题）

![tofu](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/openssh/ssh-certs-host-auth.png)

证书如何解决 TOFU 问题呢？参考下图流程

![tofu-cert](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/openssh/ssh-certs-host-certs.png)

####    OpenSSH Certificate With Ldap
[HashiCorp Vault SSH CA and Sentinel](https://medium.com/hashicorp-engineering/hashicorp-vault-ssh-ca-and-sentinel-79ea6a6960e5)

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
-   [SSH Certificate Authentication](https://docs.banyansecurity.io/docs/securing-private-resources/ssh-servers/cert-auth/)
-   [SSH Certificates: How Do OpenSSH Certificates Compare to X.509?](https://goteleport.com/blog/x509-vs-openssh-certificates/)
-   [SSH Certificates Security](https://goteleport.com/blog/ssh-certificates/)
-   [Comparing SSH Keys - RSA, DSA, ECDSA, or EdDSA?](https://goteleport.com/blog/comparing-ssh-keys/)
-   [SSH Handshake Explained](https://goteleport.com/blog/ssh-handshake-explained/)
-   [Teleport Authentication with Certificates](https://goteleport.com/docs/architecture/authentication/)

转载请注明出处，本文采用 [CC4.0](http://creativecommons.org/licenses/by-nc-nd/4.0/) 协议授权
