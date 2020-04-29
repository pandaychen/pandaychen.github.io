---
layout:     post
title:      使用 Golang 实现 SSH 和 SSHD
subtitle:   golang-ssh 库使用（一）
date:       2019-10-20
author:     pandaychen
catalog:    true
tags:
    - OpenSSH
---

##  0x00    前言
&emsp;&emsp; [golang 的 SSH 包](https://godoc.org/golang.org/x/crypto/ssh) 提供了极为丰富的接口。基于此包可以很容易的实现 SSH 客户端、SSHD 服务端以及 SSH 代理等常用工具。

##  0x01    SSH 架构
SSH 的基础架构图如下，开发的话，首先需要对 SSH 协议（分层）有个直观的认识：
![image](https://s2.ax1x.com/2019/11/05/KzgajS.png)

-   TCP 层: 建立传输层链接, 然后进行 SSH 协议处理
-   Handshake 层: SSH 协议里面的传输层, 该层主要提供加密传输
-   Authentication 层: 用户认证层 [SSH-USERAUTH], 主要提供用户认证
-   Channel && Request: SSH 协议里面的链接层 [SSH-CONNECT], 该层主要是将多个加密隧道分成逻辑通道, 可以复用通道, 常见的类型：`session`、`x11`、`forwarded-tcpip`、`direct-tcpip`, 通道里面的 Requests 是用于接收创建 ssh channel 的请求的, 而 ssh channel 就是里面的 connection, 数据的交互基于 connection 交互

##  0x02    构建 SSHD

####    SSHD 认证接口
库提供了认证接口，常使用的有下面两种方式：
-   `PasswordCallback`：可以实现 PAM、LDAP、自定义 Mysql 的认证（可以灵活实现后端认证的接口）
-   `PublicKeyCallback`：可以实现公钥认证、证书认证的接口

这里以常见的用户名 `+` 口令认证方式举例：
逻辑核心是实现 `PasswordCallback` 回调函数, 该函数的核心是返回一个 `ssh.Permissions` 对象, 其中保存了用户认证通过的信息（证书的额外信息：`CriticalOptions` 和 `Extensions`）：
```golang
type Permissions struct {
    // CriticalOptions indicate restrictions to the default
    // permissions, and are typically used in conjunction with
    // user certificates. The standard for SSH certificates
    // defines "force-command" (only allow the given command to
    // execute) and "source-address" (only allow connections from
    // the given address). The SSH package currently only enforces
    // the "source-address" critical option. It is up to server
    // implementations to enforce other critical options, such as
    // "force-command", by checking them after the SSH handshake
    // is successful. In general, SSH servers should reject
    // connections that specify critical options that are unknown
    // or not supported.
    CriticalOptions map[string]string

    // Extensions are extra functionality that the server may
    // offer on authenticated connections. Lack of support for an
    // extension does not preclude authenticating a user. Common
    // extensions are "permit-agent-forwarding",
    // "permit-X11-forwarding". The Go SSH library currently does
    // not act on any extension, and it is up to server
    // implementations to honor them. Extensions can be used to
    // pass data from the authentication callbacks to the server
    // application layer.
    Extensions map[string]string
}
```

实现口令认证的回调代码如下，如果想实现堡垒机的统一认证功能，可以实现更为负载的逻辑：）

```golang
config := &ssh.ServerConfig{
	PasswordCallback: func(c ssh.ConnMetadata, pass []byte) (*ssh.Permissions, error) {
		if c.User() == "root" && string(pass) == "rootspassword" {
			return nil, nil
		}
		return nil, fmt.Errorf("password rejected for %q", c.User())
	},
}
```

####    SSHD 服务开发
（流程图）


##  0x03    构建 SSH 客户端


##  0x04    总结

##  0x05    参考
-   [Writing a replacement to OpenSSH using Go (2/2)](https://scalingo.com/blog/writing-a-replacement-to-openssh-using-go-22.html)
-   [Simple SSH Harvester in Go](https://parsiya.net/blog/2017-12-29-simple-ssh-harvester-in-go/)
-   [Go ssh 交互式执行命令](https://mritd.me/2018/11/09/go-interactive-shell/)