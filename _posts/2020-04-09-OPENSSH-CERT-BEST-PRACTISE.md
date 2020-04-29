---
layout:     post
title:      OpenSSH Certificate 最佳实践
subtitle:
date:       2020-04-09
author:     pandaychen
header-img:
catalog: true
category:   false
tags:
    - OpenSSH
---


##  0x00    前言


##  0x01    优化 && 改造

##  0x02    证书体系的安全

####    算法安全

常用的 SSH 登录秘钥生成算法有如下四种：
-   DSA
-   RSA
-   ECDSA
-   ED25519

在安全性上，DSA 和 RSA 是易于对两个极大质数乘积做质因数分解的困难度，而 ECDSA, ED25519 则是基于椭圆曲线的离散对数难题。

总结来说：这 4 种算法的推荐排序如下：
Your SSH keys might use one of the following algorithms:

🚨 DSA: It’s unsafe and even no longer supported since OpenSSH version 7, you need to upgrade it!

⚠️ RSA: It depends on key size. If it has 3072 or 4096-bit length, then you’re good. Less than that, you probably want to upgrade it. The 1024-bit length is even considered unsafe.

👀 ECDSA: It depends on how well your machine can generate a random number that will be used to create a signature. There’s also a trustworthiness concern on the NIST curves that being used by ECDSA.

✅ Ed25519: It’s the most recommended public-key algorithm available today!

####    用户认证


####    主机认证


####    CA 密钥安全
集中化的密钥管理简化了认证管理流程，不幸的是这同时也简化了攻击面，攻击者只许获得 CA 密钥的管理就能获得对全网的访问

因此，CA 密钥的管理必须处于高安全等级，如果可能的话，将它们存储在网络无法访问的地方，并千万千万确保它们被加密了

####    证书的其他特性
由于证书的不可伪造性（Unforgeability），我们可以利用证书的内置字段或结构来提升证书使用的安全性。这里不过多介绍，开发的同学一看就明白。

主机证书：
```bash
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

用户证书：
```bash
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


-   生效时间

-   一次一密

-   一机一证

-   最小化签发

-   多个 CA 共用

##  0x03    展望


##  0x04    参考
-   [If you’re not using SSH certificates you’re doing SSH wrong](https://smallstep.com/blog/use-ssh-certificates/)
-   [Upgrade Your SSH Key to Ed25519](https://medium.com/risan/upgrade-your-ssh-key-to-ed25519-c6e8d60d3c54)
-   [使用 openssh 证书认证](https://wooyun.js.org/drops/%E4%BD%BF%E7%94%A8OpenSSH%E8%AF%81%E4%B9%A6%E8%AE%A4%E8%AF%81.html)
-   [Blessing your SSH at Lyft](https://eng.lyft.com/blessing-your-ssh-at-lyft-a1b38f81629d)
-   [Scalable and secure access with SSH](https://engineering.fb.com/security/scalable-and-secure-access-with-ssh/)
-   [Introducing the Uber SSH Certificate Authority](https://medium.com/uber-security-privacy/introducing-the-uber-ssh-certificate-authority-4f840839c5cc)
-   [Netflix-bless](https://github.com/Netflix/bless)

转载请注明出处，本文采用 [CC4.0](http://creativecommons.org/licenses/by-nc-nd/4.0/) 协议授权