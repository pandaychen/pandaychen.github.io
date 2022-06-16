---
layout: post
title: 安全：SSO 认证协议的那些事
subtitle:
date: 2021-04-01
author: pandaychen
header-img:
catalog: true
category: false
tags:
  - Authentication
  - OAuth
  - SAML
  - OpenID
---

## 0x00 前言

最近工作用到了较多的 SSO（单点登录）知识，这篇文章简单梳理下。主要涉及的到的有如下几块：

- Oauth2
- SAML2
- OpenID
- JWT
- SSH Login with SSO：如何将 SSO 的认证理念嵌入到 `OpenSSH` 的登录身份认证中去

#### 授权 && 认证

- 授权
- 认证

## 0x01 通讯主体

- 浏览器：SP 和 IdP 借助浏览器互相通信
- SP（Service Provider/Resource Server）：资源服务提供方
- IdP（Identity Provider/Authorization Server)）：身份认证服务提供方

## 0x02 OAuth2

OAuth2 是一个授权协议，有四种 Flow 流。

- OAuth2.0 使用 `HTTPS` 来做安全保护，避免了 OAuth1.0 的复杂加密，让开发人员更容易使用
- 接入的四种模式，一般采用授权码模式，比较安全

本文只介绍授权码模式（Authorization Code），其基本通信流如下图：

![normal-oauth2-flow](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/auth/sso/normal-oauth2-flow.png)

以 github 的 oauth 认证举例，其流程如下：
![github-oauth2](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/auth/sso/sso-devel-githuboauth.png)

1、客户端（如 Web 浏览器），开始请求服务端 MyService<br>
2、服务端校验 cookies，若校验失败，则重定向跳转到 IDP 认证服务 GithubApp<br>
3、IDP 认证服务展示授权页面，等待用户验证并确认授权 <br>
4、用户输入用户名 + 密码及点击确认后，授权页面请求 IDP 认证服务，获取授权码 code<br>
5、客户端获取上一步返回的授权码 code<br>
6、客户端将授权码 code 上报至服务端 MyService<br>
7、服务端 MyService 使用授权码 code 去认证服务器换取 access_token<br>
8、服务端 MyService 通过 access_token 去 IDP 认证服务获取用户基础信息，如 openid，userid 等信息 <br>
9、服务端 MyService 给用户设置 cookie，本次流程结束 <br>

关于 OAuth2 认证，需要注意的是，SP（这里是 MyService） 中设置的 `redict_url` 必须要和 IDP（这里是 github 的 App）中设置的 `Authorization callback URL` 一致，不然登录会报错。

## 0x03 SAML2

`SAML2` 的基本通信流如下（`SAML` 协议也有较多的变种，这里仅列举一种典型的）。SAML协议的核心是: **IDP和SP通过用户的浏览器的重定向访问来实现交换数据**。SP向IDP发出SAML身份认证请求消息，来请求IDP鉴别用户身份；IDP向用户索要用户名和口令（用户需要输入用户名/口令），并验证其是否正确；如果验证无误，则向SP返回SAML身份认证应答，表示该用户已经登录成功了，此外应答中里还包括一些额外的信息，来却确保应答未被篡改/伪造。

注意：上面这个流程和OAUTH协议有很大的区别，SAML2协议的最后是IDP与SP之间的交互；

![normal-saml-flow](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/auth/sso/normal-saml2-flow.png)

以 IDP Okta 的 SAML2 APP 服务验证流程为例：
![okta-saml2]()


####  SAML2的协议实例
![google-saml2](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/auth/sso/saml2-google-flow-detail.png)

## 0x04 OpenID 与 OIDC

OIDC（OpenID Connect）等于 （Identity, Authentication） + OAuth 2.0。OIDC 基于 OAuth2 协议之上构建了一个身份层的认证标准协议。OIDC 使用 OAuth2 的授权服务器来为第三方客户端提供用户的身份认证，并把对应的身份认证信息传递给客户端（如移动 APP，JS 应用等），且完全兼容 OAuth2，也就是说一个 OIDC 的服务，也可以当作一个 OAuth2 的服务来用。更通俗的说，OIDC 融合了 OpenId 的身份标识，OAuth2 的授权和 JWT 包装数据的方式。

![oidc-img]()

## 0x05 JWT（JSON Web Token）
JWT 定义了一种简洁自包含的方法用于通信双方之间以 JSON 对象的形式安全的传递信息（[官方定义](https://jwt.io/introduction)），本质是带有数字签名的格式化数据，支持多种签名方法：
- HMAC算法：常用于用户名/口令认证之后的token生成
- 非对称加密算法（如RSA/ECDSA等）：推荐，持有私钥的一方是进行签名的，公钥被分发给子系统用来验证签名，常用于APIGATEWAY等场景


举例来说：服务器认证后生成如下 JSON 对象+数字签名发回用户（为了标识该数据不可篡改，需要对此JSON进行签名）；客户端与服务端通信时依赖此JSON来认定用户身份，如下图：
```json
{
  "username": "pandaychen",
  "role": "common",
  "expired": "Thu Jun 16 16:15:12 CST 2022"
}
```

![jwt-auth](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/auth/sso/jwt-flow-2.png)


####  JWT结构
- Header：用于描述元信息，如产生 signature 的算法，`{"typ": "JWT","alg": "HS256"}`
- Payload：用于携带希望向服务端传递的信息。可以添加[官方字段](https://en.wikipedia.org/wiki/JSON_Web_Token)，例如：`iss(Issuer)`、`sub(Subject)`、`exp(Expirationtime)`，也可以添加自定义的字段
- Signature：签名，签名的算法[见](https://en.wikipedia.org/wiki/JSON_Web_Token)

![jwt-struct](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/auth/sso/sso-jwt-1.png)


## 0x06 区别

## 0x07 其他一些话题

1、能否混用 OAuth 及 SAML？<br>
2、如何设置多个 OAuth 的 CallBack Url<br>

#### SSH SSO

基于 SSH 的 SSO 实现，有两个思路：

1. 从上面对 Auth 协议的分析，如果我们在 SSH 登录前能先获取到 access-token，然后拿着此 token 调用 IDP 接口获取到用户的真实身份即可。

## 0x08 参考

- [我眼中的 SAML (Security Assertion Markup Language)](https://www.cnblogs.com/shuidao/p/3463947.html)
- [[认证 & 授权] 4. OIDC（OpenId Connect）身份认证（核心部分）](https://www.cnblogs.com/linianhui/archive/2017/05/30/openid-connect-core.html)
- [浅谈 SAML, OAuth, OpenID 和 SSO, JWT 和 Session](https://juejin.cn/post/6844903634094784520)
- [An Introduction to SAML (Security Assertion Markup Language)](https://www.secureauth.com/blog/an-introduction-to-saml-security-assertion-markup-language/)
- [What’s the Difference Between OAuth, OpenID Connect, and SAML?](https://www.okta.com/identity-101/whats-the-difference-between-oauth-openid-connect-and-saml/)
- [How OIDC Authentication Works](https://goteleport.com/blog/how-oidc-authentication-works/)
- [OAuth 2.0](https://oauth.net/2/)
- [Choosing an SSO Strategy: SAML vs OAuth2](https://www.mutuallyhuman.com/blog/choosing-an-sso-strategy-saml-vs-oauth2/)
- [OAuth & OpenID & SAML 工作流程梳理对比](https://www.jianshu.com/p/6e25a892db89)
- [Github oauth multiple authorization callback URL](https://stackoverflow.com/questions/35942009/github-oauth-multiple-authorization-callback-url)
- [I have a public key and a JWT, how do I check if it's valid in Go?](https://stackoverflow.com/questions/51834234/i-have-a-public-key-and-a-jwt-how-do-i-check-if-its-valid-in-go)
- [Diagrams of All The OpenID Connect Flows](https://darutk.medium.com/diagrams-of-all-the-openid-connect-flows-6968e3990660)
- [SAML2.0入门指南](https://www.jianshu.com/p/636c1ee16eba)
- [OpenID Connect 协议入门指南](https://www.jianshu.com/p/be7cc032a4e9)
- [OAuth2.0 协议入门指南](https://www.jianshu.com/p/6392420faf99)

转载请注明出处，本文采用 [CC4.0](http://creativecommons.org/licenses/by-nc-nd/4.0/) 协议授权
