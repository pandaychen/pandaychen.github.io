---
layout:     post
title:      中间人机制 review
subtitle:   如何优雅的实现 https mitm（透明劫持）
date:       2023-03-01
author:     pandaychen
catalog:    true
tags:
    - MITM
---


##  0x00    前言
本文探讨下 HTTPS 劫持这个话题，一些常见的网络调试工具 fiddler、charles、surge、wireshark 等，或多或少都是利用了 HTTPS 的 MITM 攻击来实现的。HTTPS劫持的核心原理是不安全的 CA（或者是非权威CA） 可以给任何网站or域名进行CA签名，TLS 服务端解密需要服务端私钥和服务段证书，然而这个不安全的 CA 可以提供用户暂时信任的服务端私钥和证书，这就好比你信任了一个信用极差的人进入你的家，这个信用极差的人可以在你的家里乱翻乱拿无恶不作。

##  0x01


####    透明代理






##  参考
-   [](https://docs.mitmproxy.org/stable/concepts-howmitmproxyworks/)