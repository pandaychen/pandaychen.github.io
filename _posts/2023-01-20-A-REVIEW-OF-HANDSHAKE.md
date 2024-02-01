---
layout:     post
title:      再看认证流程
subtitle:   TLS（HTTPS）、OpenSSH 协议的那些细节
date:       2023-01-20
author:     pandaychen
catalog:    true
tags:
    - OpenSSH
    - TLS
---


##  0x00    前言
本文梳理下 TLS（HTTPS）、OpenSSH 在握手协议上的一些细节

####  DH 协议
握手协议的共享密钥的计算基础

![DH](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/protocol/DiffieHellman.png)

##  0x01    HTTPS 认证
分为 HTTPS 单向认证和双向认证，以双向认证为例（下面第三步不验证客户端证书则为单向认证），流程如下：

![https](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/protocol/Ssl_handshake_with_two_way_authentication_with_certificates.png)

1.  第一阶段：协商，客户端发送 hello 消息，会包含自己能支持的最大支持的 TLS 版本，一个随机数，以及一系列建议的加密套件以及压缩方法；然后服务器在接收之后，也会发送 hello 消息，包含着根据客户端挑选过的 TLS 版本，加密套件以及压缩方法
2.  第二阶段：服务器证书处理，服务器会将证书发送至客户端，并且紧接着就会发送一个要求发送客户端证书的请求，客户端会在这个时间根据 CA （预埋在客户端本地存储）来验证服务器的证书
3.  第三阶段：客户端证书处理，客户端会将证书发送至服务器，服务器会根据 CA 来验证（这个 CA 可以完全是自己生成的），客户端同时会发送一个用服务器证书加密过的随机数 PreMasterSecret，然后客户端会发送一个前一个握手阶段的消息的签名来告诉服务器，客户端是的确是拥有当前证书的（有私钥），然后客户端以及服务器会根据 PreMasterSecret 以及随机数，来生成一个公共的 secret，叫做 Master secret（参考上述 DH 协议计算流程中的 `A`/`B`）
4.  第四阶段：加密通信，两边开始都用 Master secret 来进行对称的加解密

####    客户端校验证书
在 https 单向认证的场景中，客户端如何校验服务端的证书是否有效？客户端校验服务端证书的有效性主要包括以下几个步骤：

1.  检查证书有效期：客户端会检查服务端的证书是否在有效期内。证书中包含有起始日期和结束日期，如果当前日期不在这个范围内，证书将被视为无效
2.  检查证书颁发者：客户端会检查服务端证书的颁发者（Certificate Authority）是否为受信任的颁发机构，操作系统和浏览器通常会内置一份受信任的 CA 列表，如果证书颁发者不在这个列表中，证书将被视为无效（如果是 MITM 的场景，需要把由私人 CA 签发的证书预埋到客户端中）
3.  验证证书签名：客户端会使用 CA 的公钥来验证服务端证书的签名。如果签名验证失败，证书将被视为无效
4.  检查证书吊销状态：客户端可以通过查询证书吊销列表（CRL, Certificate Revocation List）或使用在线证书状态协议（OCSP, Online Certificate Status Protocol）来检查服务端证书是否被吊销。如果证书已被吊销，证书将被视为无效
5.  检查证书域名：客户端会检查服务端证书中的域名（或者通配符域名）是否与请求的域名一致。如果不一致，证书将被视为无效

如果服务端证书通过了以上所有校验，客户端会认为证书有效，从而建立安全的 HTTPS 连接。如果证书校验失败，客户端通常会显示警告信息，提示用户连接可能不安全

####    TLS1.3 的握手过程


##  0x02    OpenSSH handshake（SSH2）
OpenSSH 中包含了两套公钥，以公钥登录方式为例，先明确如下概念：
-   authorized key：用来认证，不参与 handshake 过程，是用于验证客户端身份的用户公钥
-   host-key：用来完成 handshake，是用于验证服务器身份的服务器公钥

HostKey: HostKey 是服务器的公钥，用于在客户端第一次连接到服务器时验证服务器的身份。当客户端首次尝试连接到服务器时，服务器会向客户端提供其 HostKey。客户端将此密钥的 hash 存储在 `known_hosts` 文件中，并在每次连接到服务器时使用此密钥进行验证。如果服务器的 HostKey 改变，客户端将收到警告，提示可能存在中间人攻击。这种机制保证了客户端总是连接到正确的服务器

AuthorizedKey: AuthorizedKey 是用户的公钥，存储在服务器上，用于验证尝试登录的客户端的身份。当客户端尝试使用 SSH 连接到服务器时，客户端会向服务器提供其私钥的公钥部分。服务器将此公钥与存储在 `~/.ssh/authorized_keys` 文件中的公钥进行比较。如果匹配，客户端将被授权访问服务器。这种机制允许用户使用密钥对而非密码进行身份验证，从而提高了安全性

在 OpenSSH 中，Diffie-Hellman（DH）密钥协商算法与 HostKey 有关。在客户端和服务器之间建立安全连接时，DH 密钥交换算法用于协商一个共享的会话密钥，以便对通信进行加密。DH 密钥协商与 AuthorizedKey 无关。AuthorizedKey 仅用于验证客户端身份，而不涉及到密钥协商过程


####    OpenSSH handshake flow：ECDSA
以 ECDSA 为例，OpenSSH 的握手包括如下步骤：
1.  SSH protocol version exchange：SSH 版本协商
2.  Key Exchange：客户端与服务端协商使用的公钥算法
3.  Elliptic Curve Diffie-Hellman Initialization：EDCSA 密钥初始化
4.  Elliptic Curve Diffie-Hellman Reply：ECDSA 密钥协商
5.  New Keys：生成会话密钥


![ssh-handshake](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/openssh/excellent-ssh-handshake-flow.png)

####    密钥协商过程：ECDSA
1、客户端生成 Ephemeral Key（基于 ECDH 的临时密钥对，只用于本次密钥协商，满足前向安全 forward secrecy 属性），并将其公钥在消息 secrecy 属性），并将其公钥在消息中发送给服务器 `SSH_MSG_KEX_ECDH_INIT` 中发送给服务器

![1](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/openssh/handshake/phase-1.png)

2、服务器收到 `SSH_MSG_KEX_ECDH_INIT` 报文，并在收到消息后生成自己的临时密钥对；使用客户端的公钥和自己的密钥对，服务器可以生成共享密钥 `K`（基于 DH 算法）。然后，服务器生成 `Exchange Hash H`，并对其进行签名生成 `Exchange Hash signature HS`

![2](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/openssh/handshake/SSH-Handshake-Figure2.jpg)

`Exchange Hash H` 和其签名 `Exchange Hash signature HS` 的作用如下：
-   由于 `H`/`HS` 包含共享秘密，因此证明对方能够生成共享秘密（由共享密钥 `K` 计算得出）
-   `H` 和 `HS` 的签名 / 验证循环允许客户端验证服务器拥有主机私钥的所有权，因此客户端连接到正确的服务器
-   通过对 `H` 进行签名，而不是对交换哈希的所有计算因子进行签名，效率更好

``Exchange Hash HS` 的计算因子（5 个部分）来自于如下字段：
1.  `M`，The client version, server version, clients `SSH_MSG_KEXINIT` message, servers `SSH_MSG_KEXINIT` message
2.  服务器主机公钥（或证书）HPub。该值（及其相应的私钥 HPiv）通常在进程初始化期间生成，而不是在每次握手时生成
3.  客户端公钥 `A`
4.  服务器公钥 `B`
5.  共享密钥 `K`

服务端回复 `SSH_MSG_KEX_ECDH_REPLY` 给客户端，包含服务器的临时公钥 `B`、服务器的主机公钥（Host-Key） `HPub` 以及 `HS`

3、一旦客户端收到 `SSH_MSG_KEX_ECDH_REPLY`，可以计算公钥密钥 `K` 以及 `H`，最后可以校验 `HS`

密钥交换的最后一部分是客户端从 `SSH_MSG_KEX_ECDH_REPLY` 数据中提取主机公钥（或证书） 并验证交换哈希的签名，`HS` 证明主机私钥的所有权。为了防止 OpenSSH MITM，一旦验证了签名，就会根据已知主机的本地数据库检查主机公钥（或证书）；如果此密钥（或证书）不受信任，则连接将终止

![3](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/openssh/handshake/SSH-Handshake-Figure3.jpg)

4、最后一步：生成会话密钥

客户端与服务端双方都需要生成 `6` 个密钥：两个用于加密的密钥、两个初始化向量 (`IV`) 和两个用于完整性校验的密钥，为什么不使用共享密钥 `K`？原因如下：
-   希望密钥更通用
-   避免密钥重用，保证前向安全性
-   加密密钥用于确保数据机密性，并与对称密码一起使用来加密和解密数据
-   完整性密钥通常与消息身份验证代码 (MAC) 一起使用，以确保攻击者不会操纵密文。如果不存在对密文的完整性检查，则攻击者可以操纵通过线路发送的密文，并且您可能会解密发件人未发送的内容。这种类型的方案通常称为 Encrypt-then-MAC

此外，初始化向量（`IV`） 通常是用作对称密码输入的随机数。 `IV` 的目的是确保同一消息加密两次不会产生相同的密文；最后，为什么密钥是成对出现的？如果仅使用单个完整性密钥，攻击者就可以重放客户端发送回客户端的记录，并且客户端会认为该记录有效。使用多个完整性密钥（一个用于服务器到客户端，另一个用于客户端到服务器），当客户端对密文执行完整性检查时，将会失败


5、会话密钥生成算法

-   初始 `IV` 客户端到服务器：`HASH(K || H || "A" || session_id)`
-   初始 `IV` 服务器到客户端：`HASH(K || H || "B" || session_id)`
-   客户端到服务器的加密密钥：`HASH(K || H || "C" || session_id)`
-   服务器到客户端的加密密钥：`HASH(K || H || "D" || session_id)`
-   客户端到服务器的完整性密钥：`HASH(K || H || "E" || session_id)`
-   服务器到客户端的完整性密钥：`HASH(K || H || "F" || session_id)`

一旦计算出这些值，双方都会发送通知 `SSH_MSG_NEWKEYS` 另一方密钥交换已结束，并且所有未来的通信都应使用上面生成的新密钥进行。此时，双方已就加密原语达成一致，交换秘密，并达成所选原语的密钥材料，并且可以在客户端和服务器之间建立可以提供机密性和完整性的安全通道


![4](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/openssh/handshake/SSH-Handshake-Figure4.jpg)

##  0x03    参考
-   [Client-authenticated_TLS_handshake](https://en.wikipedia.org/wiki/Transport_Layer_Security#Client-authenticated_TLS_handshake)
-   [SSH Handshake Explained](https://goteleport.com/blog/ssh-handshake-explained/)
-   [Diffie Hellman key exchange](https://en.wikipedia.org/wiki/Diffie%E2%80%93Hellman_key_exchange)
-   [OpenSSH/SSH Protocols](https://en.wikibooks.org/wiki/OpenSSH/SSH_Protocols)