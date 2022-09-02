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
  - OpenSSH
---

## 0x00 前言

最近工作用到了较多的 SSO（单点登录）知识，本文简单梳理下。主要涉及的到的有如下几块：
- OAuth2
- SAML2
- OpenID
- JWT
- OpenSSH Login with SSO：如何将 SSO 的认证理念嵌入到 `OpenSSH` 的登录身份认证中去

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
![okta-saml2](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/auth/sso/okta_saml_guidance_saml_flow.png)


####  SAML2的协议实例
![google-saml2](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/auth/sso/saml2-google-flow-detail.png)


![saml-ali](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/auth/sso_2022/aliyun-saml-flow.png)

SP 首先需要配置好 IDP 提供的 SAML 属性关键信息，如id、key等，SAML 协议主要有三个角色：
- SP（Service Provider）：向用户提供服务的web 端应用
- IDP（Identity Provide）：向SP提供用户身份信息
- User用户：通过登录IDP获取身份断言，并向SP返回身份断言来使用SP提供的服务

1.  用户请求访问 Web 应用系统
2.  Web 应用系统生成一个 SAML 身份验证请求
3.  Web 应用系统将重定向网址发送到用户的浏览器。重定向网址包含应向SSO 服务提交的编码 SAML 身份验证请求
4.  IDP 对 SAML 请求进行解码
5.  IDP对用户进行身份验证。认证成功后，IDP生成一个 SAML 响应，其中包含经过验证的用户的用户名。然后将SAML 响应编码并返回到用户的浏览器
6.  浏览器将 SAML 响应转发到 Web 应用系统 ACS URL
7.  Web 应用系统使用 IDP 的公钥验证 SAML 响应。如果成功验证该响应，ACS 则会将用户重定向到目标网址
8.  用户将重定向到目标网址并登录到 Web 应用系统

在 SAML 协议中，IDP 和 SP 不需要直接进行通讯，只要用户浏览器可以访问到 IDP 和 SP 即可。也就是说 SAML 协议在混合云环境下也可以正常进行使用，只要用户浏览器可以访问到公有云的 IDP 和内网的应用就可以使用 SAML 协议集成应用的单点登录。

## 0x04 OpenID 与 OIDC

OIDC（OpenID Connect）等于 （Identity Authentication） + OAuth 2.0。OIDC 基于 OAuth2 协议之上构建了一个身份层的认证标准协议。OIDC 使用 OAuth2 的授权服务器来为第三方客户端提供用户的身份认证，并把对应的身份认证信息传递给客户端（如移动 APP，JS 应用等），且完全兼容 OAuth2，也就是说一个 OIDC 的服务，也可以当作一个 OAuth2 的服务来用。更通俗的说，OIDC 融合了 OpenId 的身份标识，OAuth2 的授权和 JWT 包装数据的方式。

![oidc-basic](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/auth/sso_2022/oidc-basic.png)

## 0x05 JWT（JSON Web Token）
JWT 定义了一种简洁自包含的方法用于通信双方之间以 JSON 对象的形式安全的传递信息（[官方定义](https://jwt.io/introduction)），本质是带有数字签名的格式化数据，支持多种签名方法：
- HMAC算法：常用于用户名/口令认证之后的token生成
- 非对称加密算法（如RSA/ECDSA等）：推荐，持有私钥的一方是进行签名的，公钥被分发给子系统用来验证签名，常用于APIGATEWAY等场景（避免共享对此秘钥，代价是性能相对于对称加密降低），[相关代码](https://github.com/pandaychen/golang_in_action/blob/master/crypto/auth/jwt_rsa.go)


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
1、`Header`：用于描述元信息，如产生 `signature` 的算法类型等<br>
```json
{
	"typ": "JWT",
	"alg": "HS256"
}
```

2、`Payload`：用于携带希望向服务端传递的信息<br>
可以添加[官方字段](https://en.wikipedia.org/wiki/JSON_Web_Token)，例如：`iss(Issuer)`、`sub(Subject)`、`exp(Expirationtime)`，也可以添加自定义的字段

3、`Signature`：签名<br>
签名的算法[见](https://en.wikipedia.org/wiki/JSON_Web_Token)，比如，使用HMAC算法签名的方法为（注意采用的是Base64URL算法）：
```javascript
HMAC_SHA256(base64UrlEncode(header) + "." + base64UrlEncode(payload), secret)
```

![jwt-struct](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/auth/sso/sso-jwt-1.png)

####  Base64URL算法
JWT 在有些场合可能会放到 URL（比如 `api.example.com/?token=jwt-token`）。为了避免Base64 中的`3`个字符`+`、`/`和`=`对转义的影响，所以要使用Base64URL算法：
- 省略`=`
- `+`替换成`-`
- `/`替换成`_`

####  JWT的使用方式
客户端收到服务器返回的 JWT，可以储存在 Cookie 里面，也可以储存在 localStorage；客户端需要携带JWT进行通信，通常放在 HTTP 请求的头信息`Authorization`字段里面（也可以放在 `Cookie` ，但是这样不能跨域）；还有另外一种场景，若跨域的时候，可以把JWT 放在 POST 请求的数据体里面

```text
Authorization: Bearer <token>
```

1、认证：HMAC场景<br>
![jwt-hmac](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/auth/sso/jwt-auth-app-1-with-hmac.png)

2、认证：RSA场景<br>
![jwt-rsa](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/auth/sso/jwt-auth-app-2-with-rsa.png)

3、认证：基于RSA的子系统场景<br>
![jwt-rsa-subsystem](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/auth/sso/jwt-auth-for-multiple-subsystem.png)

现网中，JWT-RSA可以应用在微服务网关的认证场景，如下图。该方式避免HMAC需要共享私钥的问题，缺点是非对称秘钥的计算耗时增大。

![jwt-rsa-auth](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/auth/sso/jwt-rsa-gateway.png)

1.  由微服务网关生成RSA公私钥对并安全存储，将每个后端APP独享的公钥分发给各个应用
2.  用户请求经过微服务网关时，由网关使用对应后端应用的私钥进行JWT签名，将签名追加在HTTP头部，转发请求到后端APP
3.  后端APP使用公钥验证JWT签名是否合法

####  JWT的优缺点
- JWT 默认是不加密，但也是可以加密的。生成原始 Token 以后，可以用密钥再加密一次
- JWT 不加密的情况下，不能将秘密数据写入 JWT
- JWT 不仅可以用于认证，也可以用于交换信息。有效使用 JWT，可以降低服务器查询数据库的次数
- JWT 的最大缺点是，由于服务器不保存 session 状态，因此无法在使用过程中废止某个 token，或者更改 token 的权限。也就是说，一旦 JWT 签发了，在到期之前就会始终有效，除非服务器部署额外的逻辑
- JWT 本身包含了认证信息，一旦泄露，任何人都可以获得该令牌的所有权限。为了减少盗用，JWT 的有效期应该设置得比较短。对于一些比较重要的权限，使用时应该再次对用户进行认证
- 为了减少盗用，JWT 不应该使用 HTTP 协议明码传输，要使用 HTTPS 协议传输

## 0x06 各个认证协议的区别

####  OIDC VS OAuth2
![oidcVSauth](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/auth/sso_2022/oidc-vs-oauth.png)

## 0x07 OpenSSH With SSO

基于 OpenSSH 的 SSO 实现，有两个思路：

1. 从上面对 Auth 协议的分析，如果我们在 SSH 登录前能先获取到 access-token，然后拿着此 token 调用 IDP 接口获取到用户的真实身份即可。


## 0x08 其他一些话题

1、能否混用 OAuth 及 SAML？<br>
2、如何设置多个 OAuth 的 CallBack Url<br>

## 0x09 参考

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
- [JSON Web Token 入门教程](https://www.ruanyifeng.com/blog/2018/07/json_web_token-tutorial.html)
- [阿里云：OAuth2](https://help.aliyun.com/document_detail/174227.html)
- [阿里云：SAML](https://help.aliyun.com/document_detail/174224.html?spm=a2c4g.11186623.0.preDoc.52fe4c17U6Qifv)
- [阿里云：OIDC](https://help.aliyun.com/document_detail/174228.html)

转载请注明出处，本文采用 [CC4.0](http://creativecommons.org/licenses/by-nc-nd/4.0/) 协议授权
