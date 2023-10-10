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
##


GOOGLE/Martian 是一个开源的 HTTP/HTTPS 代理库，它可以在代理服务器上拦截和修改 HTTP/HTTPS 请求和响应，以实现一些高级功能，例如请求重定向、请求修改、响应修改、请求和响应记录等。Martian 的主要应用场景是在开发和测试过程中，帮助开发人员进行 HTTP/HTTPS 请求和响应的调试和测试。

Martian 的实现原理是通过在代理服务器上设置 HTTP/HTTPS 代理，拦截和修改 HTTP/HTTPS 请求和响应。Martian 使用 Go 语言编写，支持多种代理服务器，例如 Go 自带的 net/http、goproxy、mitmproxy 等。Martian 还支持自定义规则和过滤器，可以根据用户需求进行扩展。

Martian 的主要特点如下：

支持 HTTP/HTTPS 代理：Martian 可以拦截和修改 HTTP/HTTPS 请求和响应，实现一些高级功能。

支持多种代理服务器：Martian 支持多种代理服务器，例如 Go 自带的 net/http、goproxy、mitmproxy 等。

支持自定义规则和过滤器：Martian 支持自定义规则和过滤器，可以根据用户需求进行扩展。

支持请求重定向、请求修改、响应修改、请求和响应记录等高级功能：Martian 可以实现一些高级功能，例如请求重定向、请求修改、响应修改、请求和响应记录等。


##  0x



##  0x      MITM防护手段

##  参考
-   [](https://docs.mitmproxy.org/stable/concepts-howmitmproxyworks/)