---
layout: post
title: 使用 Golang 实现 SSH 和 SSHD（二）
subtitle: golang-ssh 库使用 && gliderlabs/ssh 分析（实战篇）
date: 2020-11-01
author: pandaychen
catalog: true
tags:
  - OpenSSH
---

## 0x00 前言
先记录一下自己实现 sshd（proxy）相关系统中需要搞懂的几个重要的 ssh 数据结构，先简单回顾下SSH协议中最核心的模块：连接协议

####	SSH 传输层协议
![ARCH](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/2019/1029-ssh.png)

再回顾下ssh协议的架构分层，详细说明下`Channel`的作用

-	传输层协议，定义了 SSH 协议数据包的格式以及 Key 交换算法
-	认证协议，定义了 SSH 协议支持的用户身份认证算法
-	SSH连接协议，定义了 SSH 支持功能特性如交互式登录会话（`session`）、TCP/IP 端口转发、X11 Forwarding等，这些功能都工作在通道 (Channel) 之上的

####	SSH连接协议
在 SSH 协议中，Channel 实现了对底层连接的多路复用（理解为虚拟连接），Channel的核心思路：

1.	通过一个数字来进行标识和区分这些 Channel
2.	实现流控 （窗口）

连接协议里的每个实际应用都是Channel，客户端与服务端双方都有可能打开Channel，大量的Channel复用同一个TCP Connection，一个Channel被双方用自己的数字标识，所以每端不同的数字可能指向的并不是相同的Channel，其他任何和Channel相关的消息都会包含对端的Channel标识

####	SSH连接协议：描述
简言之，SSH协议建立 `Channel` 的流程如下：

1、服务端和客户端任意一方，发送类型为 `SSH_MSG_CHANNEL_OPEN (90)` 的消息，通知对方需要建立 `Channel`

```TEXT
byte      SSH_MSG_CHANNEL_OPEN (90)
string    channel type, 常用可选值为: 'session', 'x11', 'forwarded-tcpip', 'direct-tcpip' 参见 https://www.rfc-editor.org/rfc/rfc4250#section-4.9.1
uint32    sender channel 编号
uint32    初始化窗口大小
uint32    最大包大小
....      下面是 channel type 特定数据
```

2、对端接收到消息后，回复类型为 `SSH_MSG_CHANNEL_OPEN_CONFIRMATION (91)` 或 `SSH_MSG_CHANNEL_OPEN_FAILURE (92)` 的消息来告知打开成功或者失败，成功定义如下：

```TEXT
byte      SSH_MSG_CHANNEL_OPEN_CONFIRMATION (91)
uint32    recipient channel 编号，这个是 SSH_MSG_CHANNEL_OPEN 中 sender channel 的值
uint32    sender channel 编号
uint32    初始化窗口大小
uint32    最大包大小
....      下面是 channel type 特定数据
```

失败定义如下：

```TEXT
byte      SSH_MSG_CHANNEL_OPEN_FAILURE (92)
uint32    recipient channel
uint32    错误码 reason code
string    描述，格式为 ISO-10646 UTF-8 encoding [RFC3629]
string    language tag [RFC3066]
```

3、`Channel` 建立完成后，在 `Channel` 中进行数据传输，主要有以下几类消息：

3.1：流量控制类消息，调节窗口大小

```TEXT
byte      SSH_MSG_CHANNEL_WINDOW_ADJUST
uint32    recipient channel
uint32    bytes to add
```

3.2：数据消息，消息的长度为 `min(数据长度, 窗口大小, 传输层协议的限制)`

3.2.1：普通数据，如交互式会话的标准输入、标准输出

```TEXT
byte      SSH_MSG_CHANNEL_DATA
uint32    recipient channel
string    data
```

3.2.2：扩展数据，如交互式会话的标准出错，标准出错对应 `data_type_code` 为 `1`，是 `data_type_code` 唯一的预定义的值

```TEXT
byte      SSH_MSG_CHANNEL_EXTENDED_DATA
uint32    recipient channel
uint32    data_type_code
string    data
```

4、最后，在打开一个特定类型的 `Channel` 后，需要对这个 `Channel` 进行 `Channel` 粒度的配置。如，建立了一个 `session` 类型的 `Channel` 后，请求对方创建一个伪终端 (`pty`、`pseudo terminal`)。这类的请求叫做 `Channel` 特定请求（Channel-Specific Requests），这类场景使用相同的数据格式如下参考。对于 `SSH_MSG_CHANNEL_REQUEST` 消息，如果 want reply 为 `true`，对方应使用 `SSH_MSG_CHANNEL_SUCCESS (98)`、`SSH_MSG_CHANNEL_FAILURE (100)` 进行回复

```text
byte      SSH_MSG_CHANNEL_REQUEST (98)
uint32    recipient channel，对方的 sender channel 编号
string    request type in US-ASCII characters only 请求类型，参见：https://www.rfc-editor.org/rfc/rfc4250#section-4.9.3
boolean   want reply 是否需要对方回复
....      下面是 request type 特定数据
```

####	Channel应用场景1：交互式会话
会话（Session）代表远程执行一个程序（广义），这个程序可能是 Shell、应用，可能有/无 tty等

1、客户端打开一个类型为 session 的 Channel

```TEXT
byte      SSH_MSG_CHANNEL_OPEN (90)
string    "session"
uint32    sender channel
uint32    initial window size
uint32    maximum packet size
```

2、服务端回复一个类型为 `SSH_MSG_CHANNEL_OPEN_CONFIRMATION` 的消息。至此 Session 类型的 Channel 创建完成

3、客户端可以请求创建一个伪终端（pty、Pseudo-Terminal）

```TEXT
byte      SSH_MSG_CHANNEL_REQUEST
uint32    recipient channel
string    "pty-req"
boolean   want_reply
string    TERM environment variable value (e.g., vt100)
uint32    terminal width, characters (e.g., 80)
uint32    terminal height, rows (e.g., 24)
uint32    terminal width, pixels (e.g., 640)
uint32    terminal height, pixels (e.g., 480)
string    encoded terminal modes
```

4、客户端可以请求设置环境变量

```TEXT
byte      SSH_MSG_CHANNEL_REQUEST
uint32    recipient channel
string    "env"
boolean   want reply
string    variable name
string    variable value
```

5、客户端启动一个 `shell`、执行一个`exec`、调用一个subsystem等（`3`选`1`）

5.1	启动 `shell`

```TEXT
byte      SSH_MSG_CHANNEL_REQUEST
uint32    recipient channel
string    "shell"
boolean   want reply
```

5.2	执行一个`exec`

```TEXT
byte      SSH_MSG_CHANNEL_REQUEST
uint32    recipient channel
string    "exec"
boolean   want reply
string    command
```

5.3	调用其他subsystem（典型如 `sftp`）

```TEXT
byte      SSH_MSG_CHANNEL_REQUEST
uint32    recipient channel
string    "subsystem"
boolean   want reply
string    subsystem name
```


6、上述的启动的程序的输入输出通过如下类型的消息传输：

6.1	标准输入、标准输出： `SSH_MSG_CHANNEL_DATA`

6.2	标准出错：`SSH_MSG_CHANNEL_EXTENDED_DATA`，扩展类型为 `SSH_EXTENDED_DATA_STDERR`

6.3	伪终端设置终端窗口大小

```TEXT
byte      SSH_MSG_CHANNEL_REQUEST
uint32    recipient channel
string    "window-change"
boolean   FALSE
uint32    terminal width, columns
uint32    terminal height, rows
uint32    terminal width, pixels
uint32    terminal height, pixels
```

6.4	信号

```TEXT
byte      SSH_MSG_CHANNEL_REQUEST
uint32    recipient channel
string    "signal"
boolean   FALSE
string    signal name (without the "SIG" prefix)
```

6.5	退出码

```TEXT
byte      SSH_MSG_CHANNEL_REQUEST
uint32    recipient channel
string    "exit-status"
boolean   FALSE
uint32    exit_status
```

6.6	退出信号

####	Channel应用场景2：TCP/IP 端口转发
SSH 本质是建立了在 client 到 server 端这两个设备之间建立了一条加密通信链路，其基于此实现了两个方向的端口转发：

-	本地转发（`direct-tcpip`）： 将 client 监听的 tcp 端口连接转发到 server 上。流量入口位于 client，因此 client 程序自身就可以自助的监听 tcp 端口，而不涉及 client 和 server 端的通讯，因此 client 监听端口不是 SSH 协议需要关心的内容，对应的指令为`ssh -L [LOCAL_IP:]LOCAL_PORT:DESTINATION:DESTINATION_PORT [USER@]SSH_SERVER`
-	远端转发（`forwarded-tcpip`）：将 server 监听的 tcp 端口连接转发到 client 上。即流量入口端口位于 server 端，因此 SSH 协议需要提供一种机制，可以让 client 告知 server 监听的 tcp 端口，对应的指令为`ssh -R [REMOTE:]REMOTE_PORT:DESTINATION:DESTINATION_PORT [USER@]SSH_SERVER`

对于每个 TCP 连接，都会创建一个 Channel

1、`direct-tcpip` 流程

1.1：client 监听一个 tcp 端口，并 accept 连接

1.2：client accept 返回后， client 发起建立一个类型为 direct-tcpip 的 Channel

```TEXT
byte      SSH_MSG_CHANNEL_OPEN
string    "direct-tcpip"
uint32    sender channel
uint32    initial window size
uint32    maximum packet size
string    host to connect
uint32    port to connect
string    originator IP address
uint32    originator port
```

1.3：server 接收到消息后，和 host to connect:port to connect TCP 端口建立 TCP 连接

1.4：至此，转发 Channel 建立完成，后续通过 `SSH_MSG_CHANNEL_DATA` 进行双向数据的转发

2、`forwarded-tcpip` 流程

2.1：准备阶段

2.1.1：client 请求 server 监听 tcp 端口，作为流量入口

```TEXT
byte      SSH_MSG_GLOBAL_REQUEST
string    "tcpip-forward"
boolean   want reply
string    address to bind (e.g., "0.0.0.0")
uint32    port number to bind
```

2.1.2	server 根据请求信息，监听对应端口，并回复：

```TEXT
byte     SSH_MSG_REQUEST_SUCCESS
uint32   port that was bound on the server
```

2.2	server accept 返回后， server 发起建立一个类型为 `direct-tcpip` 的 Channel

```TEXT
byte      SSH_MSG_CHANNEL_OPEN
string    "forwarded-tcpip"
uint32    sender channel
uint32    initial window size
uint32    maximum packet size
string    address that was connected
uint32    port that was connected
string    originator IP address
uint32    originator port
```

2.3	client 接收到消息后，和 address that was connected:port that was connected TCP 端口建立 TCP 连接

2.4	至此，转发 Channel 建立完成，后续通过 `SSH_MSG_CHANNEL_DATA` 进行双向数据的转发

转发Channel独立于Session存在，Session关闭并不意味着转发Channel也要被关闭，此外，客户端应该拒绝`direct-tcpip`请求

####	预定义名称
1、连接协议Channel类型

```TEXT
Channel type                  Reference
         ------------                  ---------
         session                       [SSH-CONNECT, Section 6.1]
         x11                           [SSH-CONNECT, Section 6.3.2]
         forwarded-tcpip               [SSH-CONNECT, Section 7.2]
         direct-tcpip                  [SSH-CONNECT, Section 7.2]
```

2、连接协议全局请求名（`SSH_MESSAGE_GLOBAL_REQUEST`）

```TEXT
Request type                  Reference
         ------------                  ---------
         tcpip-forward                 [SSH-CONNECT, Section 7.1]
         cancel-tcpip-forward          [SSH-CONNECT, Section 7.1]
```

3、连接协议Channel明细请求名（`SSH_MESSAGE_CHANNEL_REQUEST`）

```TEXT
Request type                  Reference
         ------------                  ---------
         pty-req                       [SSH-CONNECT, Section 6.2]
         x11-req                       [SSH-CONNECT, Section 6.3.1]
         env                           [SSH-CONNECT, Section 6.4]
         shell                         [SSH-CONNECT, Section 6.5]
         exec                          [SSH-CONNECT, Section 6.5]
         subsystem                     [SSH-CONNECT, Section 6.5]
         window-change                 [SSH-CONNECT, Section 6.7]
         xon-xoff                      [SSH-CONNECT, Section 6.8]
         signal                        [SSH-CONNECT, Section 6.9]
         exit-status                   [SSH-CONNECT, Section 6.10]
         exit-signal                   [SSH-CONNECT, Section 6.10]
```

## 0x01 重要结构（golang-ssh 库）

1、Channel及操作相关<br>
A Channel is an ordered, reliable, flow-controlled, duplex stream that is multiplexed over an SSH connection.（注，个人理解是 channel 最大的作用是给开发者提供了读取 SSH 明文的手段），是一种有序、可靠、流量受控的双工流，通过 SSH 连接进行多路复用

`ssh.Channel` 该interface对应一个已经建立 Channel（server 端）

```golang
type Channel interface {
	// Read reads up to len(data) bytes from the channel.
	//从 Channel 中读取数据，对应 client -> server 的 SSH_MSG_CHANNEL_DATA 消息（参考协议描述），在 session 场景对应 stdin
	Read(data []byte) (int, error)

	// Write writes len(data) bytes to the channel.
	//向 Channel 中写入数据，对应 server -> client 的 SSH_MSG_CHANNEL_DATA 消息（参见上文 channel），在 session 场景对应 stdout
	Write(data []byte) (int, error)

	// Close signals end of channel use. No data may be sent after this
	// call.
	//关闭该 channel，对应 SSH_MSG_CHANNEL_CLOSE 消息
	Close() error

	// CloseWrite signals the end of sending in-band
	// data. Requests may still be sent, and the other side may
	// still send data
	CloseWrite() error

	// SendRequest sends a channel request.  If wantReply is true,
	// it will wait for a reply and return the result as a
	// boolean, otherwise the return value will be false. Channel
	// requests are out-of-band messages so they may be sent even
	// if the data stream is closed or blocked by flow control.
	// If the channel is closed before a reply is returned, io.EOF
	// is returned.
	SendRequest(name string, wantReply bool, payload []byte) (bool, error)

	// Stderr returns an io.ReadWriter that writes to this channel
	// with the extended data type set to stderr. Stderr may
	// safely be read and written from a different goroutine than
	// Read and Write respectively.
	Stderr() io.ReadWriter
}
```

特别说下`Channel.SendRequest`这个方法，对应 `server -> client` 在该 `Channel` 上的 `SSH_MSG_CHANNEL_REQUEST`，主要这几个类型：
-	`window-change`：发送 `pty` 的 `window-change` 信息
-	`exit-status`：发送 `cmd` 的退出码消息

[`ssh.NewChannel`](https://pkg.go.dev/golang.org/x/crypto/ssh#NewChannel) 该`interface`对应一个 client 创建的 Channel 的消息（`SSH_MSG_CHANNEL_OPEN`）

```GO
type NewChannel interface {
	// Accept accepts the channel creation request. It returns the Channel
	// and a Go channel containing SSH requests. The Go channel must be
	// serviced otherwise the Channel will hang.
	Accept() (Channel, <-chan *Request, error)	//同意建立该 Channel

	// Reject rejects the channel creation request. After calling
	// this, no other methods on the Channel may be called.
	Reject(reason RejectionReason, message string) error	//拒绝建立该 Channel

	// ChannelType returns the type of the channel, as supplied by the
	// client.
	ChannelType() string

	// ExtraData returns the arbitrary payload for this channel, as supplied
	// by the client. This data is specific to the channel type.
	ExtraData() []byte
}
```

`ssh.NewChannel`的`ChannelType()`方法的主要[类型](https://www.rfc-editor.org/rfc/rfc4250#section-4.9.1)如下：

-	`session`
-	`direct-tcpip`
-	`forwarded-tcpip`
-	`auth-agent@openssh.com`
-	`x11`

####   session channel ：Request类型
While there are theoretically other types of channels possible, we currently only support session channels. The client can request channels to be opened at any time.
We currently support the following requests on the session channel. These are described in RFC 4254.


1、`env`<br>
Sets an environment variable for the soon to be executed program.

2、`pty`<br>
Requests an interactive terminal for user input.

3、`shell`<br>
Requests the default shell to be executed.

4、`exec`<br>
Requests a specific program to be executed.

5、`subsystem`<br>
Requests a well-known subsystem (e.g. sftp) to be executed.

6、`window-change`<br>
Informs the server that an interactive terminal window has changed size. This is only sent once the program has been started with the requests above.

7、`signal`<br>
Requests the server to send a signal to the currently running process.

8、`exit-status`<br>
to the client from the server when the program exits to inform the client of the exit code.

2、Request 和 Reply<br>
协议交互的收发包协议结构：

Request is a request sent outside of the normal stream of data. Requests can either be specific to an SSH channel, or they can be global.

```golang
type Request struct {
	Type      string
	WantReply bool
	Payload   []byte
	// contains filtered or unexported fields
}
```

<br>
Reply sends a response to a request. It must be called for all requests where WantReply is true and is a no-op otherwise. The payload argument is ignored for replies to channel-specific requests.<br>

```golang
//WantReply bool 字段，是否需要回复
//Payload []byte 字段，type 特定数据，可以使用 ssh.Unmarshal() 方法进行反序列化
//对 WantReply = true 的方法，必须调用该函数进行回复
func (r *Request) Reply(ok bool, payload []byte) error
```

`ssh.Request`类型， 结构体对应 `client -> server` 在该 Channel 上的 `SSH_MSG_CHANNEL_REQUEST`，其中`ssh.Request.Type`的定义，在 session channel 场景有用，主要类型：`pty-req`、`shell`、`subsystem`、`env`、`exec`等

3、核心结构 Session<br>
文档：[Session](https://pkg.go.dev/golang.org/x/crypto/ssh#Session)
A Session represents a connection to a remote command or shell.

```golang
type Session struct {
	// Stdin specifies the remote process's standard input.
	// If Stdin is nil, the remote process reads from an empty
	// bytes.Buffer.
	Stdin io.Reader

	// Stdout and Stderr specify the remote process's standard
	// output and error.
	//
	// If either is nil, Run connects the corresponding file
	// descriptor to an instance of ioutil.Discard. There is a
	// fixed amount of buffering that is shared for the two streams.
	// If either blocks it may eventually cause the remote
	// command to block.
	Stdout io.Writer
	Stderr io.Writer
	// contains filtered or unexported fields
}
```

4、Conn<br>
Conn 是一个 `interface{}`
Conn represents an SSH connection for both server and client roles. Conn is the basis for implementing an application layer, such as ClientConn, which implements the traditional shell access for clients.<br>

```golang
type Conn interface {
	ConnMetadata

	// SendRequest sends a global request, and returns the
	// reply. If wantReply is true, it returns the response status
	// and payload. See also RFC4254, section 4.
	SendRequest(name string, wantReply bool, payload []byte) (bool, []byte, error)

	// OpenChannel tries to open an channel. If the request is
	// rejected, it returns *OpenChannelError. On success it returns
	// the SSH Channel and a Go channel for incoming, out-of-band
	// requests. The Go channel must be serviced, or the
	// connection will hang.
	OpenChannel(name string, data []byte) (Channel, <-chan *Request, error)

	// Close closes the underlying network connection
	Close() error

	// Wait blocks until the connection has shut down, and returns the
	// error causing the shutdown.
	Wait() error
}
```

5、ConnMetadata<br>
ConnMetadata holds metadata for the connection.<br>

```golang
type ConnMetadata interface {
	// User returns the user ID for this connection.
	User() string

	// SessionID returns the session hash, also denoted by H.
	SessionID() []byte

	// ClientVersion returns the client's version string as hashed
	// into the session ID.
	ClientVersion() []byte

	// ServerVersion returns the server's version string as hashed
	// into the session ID.
	ServerVersion() []byte

	// RemoteAddr returns the remote address for this connection.
	RemoteAddr() net.Addr

	// LocalAddr returns the local address for this connection.
	LocalAddr() net.Addr
}
```

6、`ServerConn`

ServerConn is an authenticated SSH connection, as seen from the server<br>

```golang
type ServerConn struct {
	Conn

	// If the succeeding authentication callback returned a
	// non-nil Permissions pointer, it is stored here.
	Permissions *Permissions
}
```

7、Session<br>
A Session represents a connection to a remote command or shell.
```golang
type Session struct {
	// Stdin specifies the remote process's standard input.
	// If Stdin is nil, the remote process reads from an empty
	// bytes.Buffer.
	Stdin io.Reader

	// Stdout and Stderr specify the remote process's standard
	// output and error.
	//
	// If either is nil, Run connects the corresponding file
	// descriptor to an instance of ioutil.Discard. There is a
	// fixed amount of buffering that is shared for the two streams.
	// If either blocks it may eventually cause the remote
	// command to block.
	Stdout io.Writer
	Stderr io.Writer
	// contains filtered or unexported fields
}
```

## 0x02 认证、配置相关

1、Signer<br>
A Signer can create signatures that verify against a public key.<br>

```golang
type Signer interface {
	// PublicKey returns an associated PublicKey instance.
	PublicKey() PublicKey

	// Sign returns raw signature for the given data. This method
	// will apply the hash specified for the keytype to the data.
	Sign(rand io.Reader, data []byte) (*Signature, error)
}
```

2、PublicKey<br>
PublicKey is an abstraction of different types of public keys.<br>

```golang
type PublicKey interface {
	// Type returns the key's type, e.g."ssh-rsa".
	Type() string

	// Marshal returns the serialized key data in SSH wire format,
	// with the name prefix. To unmarshal the returned data, use
	// the ParsePublicKey function.
	Marshal() []byte

	// Verify that sig is a signature on the given data using this
	// key. This function will hash the data appropriately first.
	Verify(data []byte, sig *Signature) error
}
```

3、CertChecker<br>
CertChecker does the work of verifying a certificate. Its methods can be plugged into ClientConfig.HostKeyCallback and ServerConfig.PublicKeyCallback. For the CertChecker to work, minimally, the IsAuthority callback should be set.

```golang
type CertChecker struct {
	// SupportedCriticalOptions lists the CriticalOptions that the
	// server application layer understands. These are only used
	// for user certificates.
	SupportedCriticalOptions []string

	// IsUserAuthority should return true if the key is recognized as an
	// authority for the given user certificate. This allows for
	// certificates to be signed by other certificates. This must be set
	// if this CertChecker will be checking user certificates.
	IsUserAuthority func(auth PublicKey) bool

	// IsHostAuthority should report whether the key is recognized as
	// an authority for this host. This allows for certificates to be
	// signed by other keys, and for those other keys to only be valid
	// signers for particular hostnames. This must be set if this
	// CertChecker will be checking host certificates.
	IsHostAuthority func(auth PublicKey, address string) bool

	// Clock is used for verifying time stamps. If nil, time.Now
	// is used.
	Clock func() time.Time

	// UserKeyFallback is called when CertChecker.Authenticate encounters a
	// public key that is not a certificate. It must implement validation
	// of user keys or else, if nil, all such keys are rejected.
	UserKeyFallback func(conn ConnMetadata, key PublicKey) (*Permissions, error)

	// HostKeyFallback is called when CertChecker.CheckHostKey encounters a
	// public key that is not a certificate. It must implement host key
	// validation or else, if nil, all such keys are rejected.
	HostKeyFallback HostKeyCallback

	// IsRevoked is called for each certificate so that revocation checking
	// can be implemented. It should return true if the given certificate
	// is revoked and false otherwise. If nil, no certificates are
	// considered to have been revoked.
	IsRevoked func(cert *Certificate) bool
}
```

4、ClientConfig<br>
A ClientConfig structure is used to configure a Client. It must not be modified after having been passed to an SSH function.<br>

```golang
type ClientConfig struct {
	// Config contains configuration that is shared between clients and
	// servers.
	Config

	// User contains the username to authenticate as.
	User string

	// Auth contains possible authentication methods to use with the
	// server. Only the first instance of a particular RFC 4252 method will
	// be used during authentication.
	Auth []AuthMethod

	// HostKeyCallback is called during the cryptographic
	// handshake to validate the server's host key. The client
	// configuration must supply this callback for the connection
	// to succeed. The functions InsecureIgnoreHostKey or
	// FixedHostKey can be used for simplistic host key checks.
	HostKeyCallback HostKeyCallback

	// BannerCallback is called during the SSH dance to display a custom
	// server's message. The client configuration can supply this callback to
	// handle it as wished. The function BannerDisplayStderr can be used for
	// simplistic display on Stderr.
	BannerCallback BannerCallback

	// ClientVersion contains the version identification string that will
	// be used for the connection. If empty, a reasonable default is used.
	ClientVersion string

	// HostKeyAlgorithms lists the key types that the client will
	// accept from the server as host key, in order of
	// preference. If empty, a reasonable default is used. Any
	// string returned from PublicKey.Type method may be used, or
	// any of the CertAlgoXxxx and KeyAlgoXxxx constants.
	HostKeyAlgorithms []string

	// Timeout is the maximum amount of time for the TCP connection to establish.
	//
	// A Timeout of zero means no timeout.
	Timeout time.Duration
}
```

5、ServerConfig<br>
ServerConfig holds server specific configuration data.<br>

```golang
type ServerConfig struct {
	// Config contains configuration shared between client and server.
	Config

	// NoClientAuth is true if clients are allowed to connect without
	// authenticating.
	NoClientAuth bool

	// MaxAuthTries specifies the maximum number of authentication attempts
	// permitted per connection. If set to a negative number, the number of
	// attempts are unlimited. If set to zero, the number of attempts are limited
	// to 6.
	MaxAuthTries int

	// PasswordCallback, if non-nil, is called when a user
	// attempts to authenticate using a password.
	PasswordCallback func(conn ConnMetadata, password []byte) (*Permissions, error)

	// PublicKeyCallback, if non-nil, is called when a client
	// offers a public key for authentication. It must return a nil error
	// if the given public key can be used to authenticate the
	// given user. For example, see CertChecker.Authenticate. A
	// call to this function does not guarantee that the key
	// offered is in fact used to authenticate. To record any data
	// depending on the public key, store it inside a
	// Permissions.Extensions entry.
	PublicKeyCallback func(conn ConnMetadata, key PublicKey) (*Permissions, error)

	// KeyboardInteractiveCallback, if non-nil, is called when
	// keyboard-interactive authentication is selected (RFC
	// 4256). The client object's Challenge function should be
	// used to query the user. The callback may offer multiple
	// Challenge rounds. To avoid information leaks, the client
	// should be presented a challenge even if the user is
	// unknown.
	KeyboardInteractiveCallback func(conn ConnMetadata, client KeyboardInteractiveChallenge) (*Permissions, error)

	// AuthLogCallback, if non-nil, is called to log all authentication
	// attempts.
	AuthLogCallback func(conn ConnMetadata, method string, err error)

	// ServerVersion is the version identification string to announce in
	// the public handshake.
	// If empty, a reasonable default is used.
	// Note that RFC 4253 section 4.2 requires that this string start with
	// "SSH-2.0-".
	ServerVersion string

	// BannerCallback, if present, is called and the return string is sent to
	// the client after key exchange completed but before authentication.
	BannerCallback func(conn ConnMetadata) string

	// GSSAPIWithMICConfig includes gssapi server and callback, which if both non-nil, is used
	// when gssapi-with-mic authentication is selected (RFC 4462 section 3).
	GSSAPIWithMICConfig *GSSAPIWithMICConfig
	// contains filtered or unexported fields
}
```

##	0x03	gliderlabs/ssh 库实现分析
这个库封装了 golang-ssh 的接口，使得可以通过该库直接开发需要的 ssh 应用，如 sshd、ssh 交互式命令行等等。此外，该库在重要的位置都预留了用户钩子，开发者可以将自己的逻辑内嵌进去。

####	代码组织
-	[server.go](https://github.com/gliderlabs/ssh/blob/master/server.go)：封装了 sshd 实现相关的结构及接口
-	[session.go](https://github.com/gliderlabs/ssh/blob/master/session.go)：封装了 ssh-session 相关的结构及接口
-	[ssh.go](https://github.com/gliderlabs/ssh/blob/master/ssh.go)：提供了供用户调用的外部接口，如 `Serve`、`ListenAndServe` 等等
-	[tcpip.go](https://github.com/gliderlabs/ssh/blob/master/tcpip.go)：提供了端口转发的实现，以及 `DirectTCPIPHandler` 方法（可直接调用）
-	[agent.go](https://github.com/gliderlabs/ssh/blob/master/agent.go)：提供了 ssh-agent 相关的接口及实现
-	[context.go](https://github.com/gliderlabs/ssh/blob/master/context.go)：封装 `context.Context`，加入 ssh 的属性，使得每条连接都关联一个唯一的 context


####	Session 结构
`ssh.Session` 抽象了一条 ssh 会话，以及该会话可以获取到的所有关联的属性
```golang
type Session interface {
	gossh.Channel

	// User returns the username used when establishing the SSH connection.
	User() string

	// RemoteAddr returns the net.Addr of the client side of the connection.
	RemoteAddr() net.Addr

	// LocalAddr returns the net.Addr of the server side of the connection.
	LocalAddr() net.Addr

	// Environ returns a copy of strings representing the environment set by the
	// user for this session, in the form "key=value".
	Environ() []string

	// Exit sends an exit status and then closes the session.
	Exit(code int) error

	// Command returns a shell parsed slice of arguments that were provided by the
	// user. Shell parsing splits the command string according to POSIX shell rules,
	// which considers quoting not just whitespace.
	Command() []string

	// RawCommand returns the exact command that was provided by the user.
	RawCommand() string

	// Subsystem returns the subsystem requested by the user.
	Subsystem() string

	// PublicKey returns the PublicKey used to authenticate. If a public key was not
	// used it will return nil.
	PublicKey() PublicKey

	// Context returns the connection's context. The returned context is always
	// non-nil and holds the same data as the Context passed into auth
	// handlers and callbacks.
	//
	// The context is canceled when the client's connection closes or I/O
	// operation fails.
	Context() Context

	// Permissions returns a copy of the Permissions object that was available for
	// setup in the auth handlers via the Context.
	Permissions() Permissions

	// Pty returns PTY information, a channel of window size changes, and a boolean
	// of whether or not a PTY was accepted for this session.
	Pty() (Pty, <-chan Window, bool)

	// Signals registers a channel to receive signals sent from the client. The
	// channel must handle signal sends or it will block the SSH request loop.
	// Registering nil will unregister the channel from signal sends. During the
	// time no channel is registered signals are buffered up to a reasonable amount.
	// If there are buffered signals when a channel is registered, they will be
	// sent in order on the channel immediately after registering.
	Signals(c chan<- Signal)

	// Break regisers a channel to receive notifications of break requests sent
	// from the client. The channel must handle break requests, or it will block
	// the request handling loop. Registering nil will unregister the channel.
	// During the time that no channel is registered, breaks are ignored.
	Break(c chan<- bool)
}
```

上面公共接口对应的实例化实现如下：
```golang
type session struct {
	sync.Mutex
	gossh.Channel
	conn              *gossh.ServerConn	// 指向上层的连接
	handler           Handler
	subsystemHandlers map[string]SubsystemHandler
	handled           bool
	exited            bool
	pty               *Pty
	winch             chan Window
	env               []string
	ptyCb             PtyCallback
	sessReqCb         SessionRequestCallback
	rawCmd            string
	subsystem         string
	ctx               Context
	sigCh             chan<- Signal
	sigBuf            []Signal
	breakCh           chan<- bool
}
```

这里有个比较重要的成员 `gossh.Channel`，见第一节，该数据结构是 SSH 数据层面通信的核心结构（对应原生库的实例化 [结构](https://cs.opensource.google/go/x/crypto/+/793ad666:ssh/channel.go;l=150) 是 `channel`），其核心方法如下：
-	`SendRequest`：request 类型的请求发送
-	`Read`：读取明文数据
-	`Write`：写入明文数据

封装 `gossh.Channel` 的意义是，一是可以直接调用 `gossh.Channel` 暴露的方法来完成 ssh 数据层面的转发 / 发送；二是可以基于此来实现 ssh 会话数据的劫持（即 SSH 会话审计）

####	默认的 session 处理方法：DefaultSessionHandler
项目提供了默认的 ssh session 处理方法 `DefaultSessionHandler`，在 `DefaultSessionHandler` 方法中，只提供了常见的 `reqeusts` 的处理，并未提供 `pty/tty` 之类的对接（需要由开发者自行实现），比如：
-	基于 `top` 命令的带 tty 的命令行 [实现](https://github.com/pandaychen/openssh-magic-change/blob/master/sshd/top.go)
-	基于 `/bin/bash` 的 bash 命令交互式 [实现](https://github.com/pandaychen/openssh-magic-change/blob/master/sshd/tty.go)

```golang
func DefaultSessionHandler(srv *Server, conn *gossh.ServerConn, newChan gossh.NewChannel, ctx Context) {
	ch, reqs, err := newChan.Accept()		// 获取 ssh 的 channel 及 requests
	if err != nil {
		return
	}
	sess := &session{
		Channel:           ch,
		conn:              conn,
		handler:           srv.Handler,
		ptyCb:             srv.PtyCallback,
		sessReqCb:         srv.SessionRequestCallback,
		subsystemHandlers: srv.SubsystemHandlers,
		ctx:               ctx,
	}
	sess.handleRequests(reqs)
}
```

`handleRequests` 方法，提供了不少标准协议的 `requests` 类型实现，参见 RFC[文档](https://datatracker.ietf.org/doc/html/rfc4250#section-4.9.3)，这里注意两点：
1.	数据需要按照 ssh 的协议格式编码，如 `gossh.Unmarshal`
2.	需要应答的请求，一定要调用 `req.Reply` 进行响应

```golang
func (sess *session) handleRequests(reqs <-chan *gossh.Request) {
	for req := range reqs {
		switch req.Type {
		case "shell", "exec":
			if sess.handled {
				req.Reply(false, nil)
				continue
			}

			var payload = struct{Value string}{}
			gossh.Unmarshal(req.Payload, &payload)
			sess.rawCmd = payload.Value

			// If there's a session policy callback, we need to confirm before
			// accepting the session.
			if sess.sessReqCb != nil && !sess.sessReqCb(sess, req.Type) {
				sess.rawCmd = ""
				req.Reply(false, nil)
				continue
			}

			sess.handled = true
			req.Reply(true, nil)

			go func() {
				sess.handler(sess)
				sess.Exit(0)
			}()
		case "subsystem":
			if sess.handled {
				req.Reply(false, nil)
				continue
			}

			var payload = struct{Value string}{}
			gossh.Unmarshal(req.Payload, &payload)
			sess.subsystem = payload.Value

			// If there's a session policy callback, we need to confirm before
			// accepting the session.
			if sess.sessReqCb != nil && !sess.sessReqCb(sess, req.Type) {
				sess.rawCmd = ""
				req.Reply(false, nil)
				continue
			}

			handler := sess.subsystemHandlers[payload.Value]
			if handler == nil {
				handler = sess.subsystemHandlers["default"]
			}
			if handler == nil {
				req.Reply(false, nil)
				continue
			}

			sess.handled = true
			req.Reply(true, nil)

			go func() {
				handler(sess)
				sess.Exit(0)
			}()
		case "env":
			if sess.handled {
				req.Reply(false, nil)
				continue
			}
			var kv struct{Key, Value string}
			gossh.Unmarshal(req.Payload, &kv)
			sess.env = append(sess.env, fmt.Sprintf("%s=%s", kv.Key, kv.Value))
			req.Reply(true, nil)
		case "signal":
			var payload struct{Signal string}
			gossh.Unmarshal(req.Payload, &payload)
			sess.Lock()
			if sess.sigCh != nil {
				sess.sigCh <- Signal(payload.Signal)
			} else {
				if len(sess.sigBuf) < maxSigBufSize {
					sess.sigBuf = append(sess.sigBuf, Signal(payload.Signal))
				}
			}
			sess.Unlock()
		case "pty-req":
			if sess.handled || sess.pty != nil {
				req.Reply(false, nil)
				continue
			}
			ptyReq, ok := parsePtyRequest(req.Payload)
			if !ok {
				req.Reply(false, nil)
				continue
			}
			if sess.ptyCb != nil {
				ok := sess.ptyCb(sess.ctx, ptyReq)
				if !ok {
					req.Reply(false, nil)
					continue
				}
			}
			sess.pty = &ptyReq
			sess.winch = make(chan Window, 1)
			sess.winch <- ptyReq.Window
			defer func() {
				// when reqs is closed
				close(sess.winch)
			}()
			req.Reply(ok, nil)
		case "window-change":
			if sess.pty == nil {
				req.Reply(false, nil)
				continue
			}
			win, ok := parseWinchRequest(req.Payload)
			if ok {
				sess.pty.Window = win
				sess.winch <- win
			}
			req.Reply(ok, nil)
		case agentRequestType:
			// TODO: option/callback to allow agent forwarding
			SetAgentRequested(sess.ctx)
			req.Reply(true, nil)
		case "break":
			ok := false
			sess.Lock()
			if sess.breakCh != nil {
				sess.breakCh <- true
				ok = true
			}
			req.Reply(ok, nil)
			sess.Unlock()
		default:
			// TODO: debug log
			req.Reply(false, nil)
		}
	}
}
```

##	0x04	subsystem：子系统
如何理解 sshd 中的 subsytem 机制？subsystem 是 SSH 协议的一个扩展功能，它允许 SSH 服务器在远程客户端的请求下，启动特定的预定义服务。这些服务可以是文件传输、版本控制系统、远程桌面等。subsytem 是一种 **在 SSH 会话中运行特定应用程序的方法，而无需在远程计算机上启动一个完整的 shell 会话**。`sftp-server` 就是典型的 subsytem 服务，在 `/etc/ssh/sshd_config` 一般可以看到 `Subsystem sftp /usr/libexec/openssh/sftp-server` 的配置选项，当客户端发起一个 SFTP 会话时，sshd 将启动 `/usr/lib/openssh/sftp-server` 程序，并将其与客户端的 SSH 会话关联。

此外，还有几种典型的场景：
- git server：`Subsystem git /usr/bin/git-shell`
- x11 转发：`Subsystem x11 /usr/bin/Xvnc`

子系统的优势是可以为特定任务提供专用的环境。如使用 SFTP 子系统，用户可以在不启动一个完整的远程 shell 会话的情况下，安全地传输文件。这有助于提高安全性，同时减少了系统资源的消耗

#### 	gliderlabs/ssh 的子系统
前文分析 `handleRequests` 这个核心方法中，已经包含了针对 `subsytem` 的 [实现](https://github.com/gliderlabs/ssh/blob/master/session.go#L264)：

```GO
func (sess *session) handleRequests(reqs <-chan *gossh.Request) {
	for req := range reqs {
		switch req.Type {
			//...
		case "subsystem":
		    // 调用用户传入的 handler
            handler := sess.subsystemHandlers[payload.Value]
			if handler == nil {
				handler = sess.subsystemHandlers["default"]
			}
			if handler == nil {
				req.Reply(false, nil)
				continue
			}

			sess.handled = true
			req.Reply(true, nil)

			go func() {
				handler(sess)
				sess.Exit(0)
			}()
		}
	}
}
```

再看下 `sftp-server` 的 [实现](https://github.com/gliderlabs/ssh/blob/master/_examples/ssh-sftpserver/sftp.go)，这里注意 `SftpHandler` 的实现，其启动了一个 `sftp-server`，该 server 基于 `github.com/pkg/sftp` 构建

```GO
// SftpHandler handler for SFTP subsystem
func SftpHandler(sess ssh.Session) {
	debugStream := io.Discard
	serverOptions := []sftp.ServerOption{
		sftp.WithDebug(debugStream),
	}
	server, err := sftp.NewServer(
		sess,   // 将 ssh.Session 类型传入参数，ssh.Session 包含了 gossh.Channel，可以进行 ssh 的明文流数据读写
		serverOptions...,
	)
	if err != nil {
		log.Printf("sftp server init error: %s\n", err)
		return
	}
	if err := server.Serve(); err == io.EOF {
		server.Close()
		fmt.Println("sftp client exited session.")
	} else if err != nil {
		fmt.Println("sftp server completed with error:", err)
	}
}

func main() {
	ssh_server := ssh.Server{
		Addr: "127.0.0.1:2222",
		SubsystemHandlers: map[string]ssh.SubsystemHandler{
			"sftp": SftpHandler,
		},
	}
	log.Fatal(ssh_server.ListenAndServe())
}
```

最后再看下 `github.com/pkg/sftp` 的启动 [相关代码](https://github.com/pkg/sftp/blob/master/server.go#L92)`：

```go
// NewServer creates a new Server instance around the provided streams, serving
// content from the root of the filesystem.  Optionally, ServerOption
// functions may be specified to further configure the Server.
//
// A subsequent call to Serve() is required to begin serving files over SFTP.
func NewServer(rwc io.ReadWriteCloser, options ...ServerOption) (*Server, error) {
	svrConn := &serverConn{
		conn: conn{
			Reader:      rwc,  // 关联 ssh session 输出
			WriteCloser: rwc,  // 关联 ssh session 输入
		},
	}
	s := &Server{
		serverConn:  svrConn,
		debugStream: ioutil.Discard,
		pktMgr:      newPktMgr(svrConn),
		openFiles:   make(map[string]*os.File),
		maxTxPacket: defaultMaxTxPacket,
	}

	for _, o := range options {
		if err := o(s); err != nil {
			return nil, err
		}
	}

	return s, nil
}

// Serve serves SFTP connections until the streams stop or the SFTP subsystem
// is stopped. It returns nil if the server exits cleanly.
func (svr *Server) Serve() error {
	defer func() {
		if svr.pktMgr.alloc != nil {
			svr.pktMgr.alloc.Free()
		}
	}()
	var wg sync.WaitGroup

	// 独立的goroutine，处理一组sftp请求
	runWorker := func(ch chan orderedRequest) {
		wg.Add(1)
		go func() {
			defer wg.Done()
			if err := svr.sftpServerWorker(ch); err != nil {
				svr.conn.Close() // shuts down recvPacket
			}
		}()
	}

	// 重要：pktChan是生产者消费者的通信通道
	pktChan := svr.pktMgr.workerChan(runWorker)

	var err error
	var pkt requestPacket
	var pktType uint8
	var pktBytes []byte
	for {
		// 从 ssh 会话接收数据
		pktType, pktBytes, err = svr.serverConn.recvPacket(svr.pktMgr.getNextOrderID())
		if err != nil {
			// Check whether the connection terminated cleanly in-between packets.
			if err == io.EOF {
				err = nil
			}
			// we don't care about releasing allocated pages here, the server will quit and the allocator freed
			break
		}

		pkt, err = makePacket(rxPacket{fxp(pktType), pktBytes})
		if err != nil {
			switch {
			case errors.Is(err, errUnknownExtendedPacket):
				//if err := svr.serverConn.sendError(pkt, ErrSshFxOpUnsupported); err != nil {
				//	debug("failed to send err packet: %v", err)
				//	svr.conn.Close() // shuts down recvPacket
				//	break
				//}
			default:
				debug("makePacket err: %v", err)
				svr.conn.Close() // shuts down recvPacket
				break
			}
		}

		//这里是sftp服务端处理的核心机制
		//将接收到的 SFTP 协议数据包有序地分发给工作协程进行处理，确保 SFTP 协议的完整性和顺序性
		//newOrderedRequest方法用于创建有序请求，包含请求包数据（pkt）以及顺序ID（保证请求按顺序处理）

		// 重要：生产者向pktChan添加数据
		pktChan <- svr.pktMgr.newOrderedRequest(pkt)
	}

	close(pktChan) // shuts down sftpServerWorkers
	wg.Wait()      // wait for all workers to exit

	// close any still-open files
	for handle, file := range svr.openFiles {
		fmt.Fprintf(svr.debugStream, "sftp server file with handle %q left open: %v\n", handle, file.Name())
		file.Close()
	}
	return err // error from recvPacket
}
```

####	sftp服务端的并发模型
sftp服务端实现的并发模型，基本上还是生产消费组模式（一生产加两级消费者）

```TEXT
主接收循环 （生产者）
     | （产生请求）
`pktChan` （缓冲通道） 
     | (分发请求)
多个`sftpServerWorker` （消费者组），又分为`rwChan`与`cmdChan`
     | （并行处理）
独立处理每个请求
```

上面的`Serve`方法中，定义了消费者的处理函数`runWorker`：

```GO
runWorker := func(ch chan orderedRequest) {
	wg.Add(1)
	go func() {
		defer wg.Done()
		if err := svr.sftpServerWorker(ch); err != nil {
			svr.conn.Close() // shuts down recvPacket
		}
	}()
}

//初始化消费者组
pktChan := svr.pktMgr.workerChan(runWorker)
```

`workerChan`方法用于初始化消费者组，创建生产者/消费者的通信管道，根据协议拆分为两类worker：

-	读写：关联`rwChan`，会根据cpu核心数启动对应个数的goroutine
-	命令字：关联`cmdChan`，启动`1`个

同时，`workerChan`也承担了消费者的功能，将`pktChan`中由生产者发送的数据做预处理，根据sftp协议的消息类型（上面两种），将包分别转发给响应的读写或者命令字goroutine处理

```go
//https://github.com/pkg/sftp/blob/master/packet-manager.go#L115
// Passed a worker function, returns a channel for incoming packets.
// Keep process packet responses in the order they are received while
// maximizing throughput of file transfers.
func (s *packetManager) workerChan(runWorker func(chan orderedRequest),
) chan orderedRequest {
	// multiple workers for faster read/writes
	rwChan := make(chan orderedRequest, SftpServerWorkerCount)
	for i := 0; i < SftpServerWorkerCount; i++ {
		runWorker(rwChan)
	}

	// single worker to enforce sequential processing of everything else
	cmdChan := make(chan orderedRequest)
	runWorker(cmdChan)

	//创建缓冲区
	pktChan := make(chan orderedRequest, SftpServerWorkerCount)
	go func() {
		for pkt := range pktChan {
			switch pkt.requestPacket.(type) {
			case *sshFxpReadPacket, *sshFxpWritePacket:
				s.incomingPacket(pkt)
				rwChan <- pkt
				continue
			case *sshFxpClosePacket:
				// wait for reads/writes to finish when file is closed
				// incomingPacket() call must occur after this
				s.working.Wait()
			}
			s.incomingPacket(pkt)
			// all non-RW use sequential cmdChan
			cmdChan <- pkt
		}
		close(rwChan)
		close(cmdChan)
		s.close()
	}()

	return pktChan
}
```

#### 	客户端调用子系统
参考sftp库的客户端[`NewClient`](https://github.com/pkg/sftp/blob/v1.13.10/client.go#L197)实现：

```GO
func NewClient(conn *ssh.Client, opts ...ClientOption) (*Client, error) {
	s, err := conn.NewSession()
	if err != nil {
		return nil, err
	}

	pw, err := s.StdinPipe()
	if err != nil {
		return nil, err
	}
	pr, err := s.StdoutPipe()
	if err != nil {
		return nil, err
	}
	perr, err := s.StderrPipe()
	if err != nil {
		return nil, err
	}

	//客户端调用sftp子系统
	if err := s.RequestSubsystem("sftp"); err != nil {
		return nil, err
	}

	//初始化sftp的管道
	/*
	rd io.Reader： 从服务器读取数据的管道（SSH会话的stdout）
	stderr io.Reader： 服务器的错误输出管道（SSH会话的stderr）
	wr io.WriteCloser： 向服务器写入数据的管道（SSH会话的stdin）
	wait func() error：等待远程命令结束的函数
	opts： 客户端配置选项
	*/
	return newClientPipe(pr, perr, pw, s.Wait, opts...)
}
```

##	0x05	sftp与SSH的分层设计与连接管理
从sftp客户端的实现中，可以了解到在这种分层协议代码设计上的模式


####	连接设计
连接设计，代码[实现](https://github.com/pkg/sftp/blob/master/conn.go)，包含如下三层实现，是非常经典的请求-响应匹配机制实现

```TEXT
clientConn (高级连接管理)
    |
   conn (基础数据包传输)  
    |
 底层传输 (SSH/TCP等)
```

```go
type conn struct {
	io.Reader			// 从服务器读取数据
	io.WriteCloser		// 向服务器写入数据
	// this is the same allocator used in packet manager
	alloc      *allocator

	// 保护发送操作的互斥锁
	sync.Mutex // used to serialise writes to sendPacket
}

type clientConn struct {
	conn		//受mutex保护发送操作
	wg sync.WaitGroup

	wait func() error // if non-nil, call this during Wait() to get a possible remote status error.

	// 保护inflight的访问
	sync.Mutex         	                 // protects inflight
	inflight   map[uint32]chan<- result	// 请求跟踪

	closed chan struct{}			// 连接关闭信号
	err    error
}

type Client struct {
	clientConn		//连接管理（并发模式）

	stderrTo io.Writer

	ext map[string]string // Extensions (name -> data).

	maxPacket             int // max packet size read or written.
	maxConcurrentRequests int
	nextid                uint32

	// write concurrency is… error prone.
	// Default behavior should be to not use it.
	useConcurrentWrites    bool
	useFstat               bool
	disableConcurrentReads bool
}
```

`conn`是基础数据包传输层的结构：

```GO
// 发送数据包（线程安全）
func (c *conn) sendPacket(m encoding.BinaryMarshaler) error {
    c.Lock()         // 确保同一时间只有一个发送操作
    defer c.Unlock()	// 保护底层网络写入的互斥锁，确保同一时间只有一个包在发送
    
    return sendPacket(c, m)  // 实际的网络io操作
}

// 接收数据包
func (c *conn) recvPacket(orderID uint32) (fxp, []byte, error) {
    return recvPacket(c, c.alloc, orderID)
}
```

`clientConn`表示客户端连接，管理未完成的请求和连接状态，核心成员`inflightmap`用来管理所有已发送但尚未收到响应的请求

-	`key`： 请求ID，唯一标识每个发出的请求
-	`value`为`chan<- result`：结果通道，用于接收该请求对应的响应

其中在初始化sftp的`Client`结构时，会同时初始化`clientConn`

```GO
c := &Client{
    clientConn: clientConn{
        conn: conn{
            Reader:      rd,  // 从服务器读取响应
            WriteCloser: wr,  // 向服务器发送请求
        },
        inflight: make(map[uint32]chan<- result), // 正在处理的请求映射
        closed:   make(chan struct{}),            // 连接关闭信号
        wait:     wait,                           // 远程命令等待函数
    },
    ext: make(map[string]string), // 扩展功能支持映射
    
    // 默认配置
    maxPacket:             1 << 15,              // 最大数据包大小(32KB)
    maxConcurrentRequests: 64,                   // 最大并发请求数
}
```

####	核心工作机制解析
1、请求-响应匹配流程

```GO
// 完整的请求-响应生命周期
func (c *clientConn) sendPacket(ctx context.Context, ch chan result, p idmarshaler) (fxp, []byte, error) {
    // 1. 准备响应通道
    if cap(ch) < 1 {
        ch = make(chan result, 1)  // 缓冲大小为1，避免阻塞
    }
    
    // 2. 分发请求（注册到inflight映射 + 发送数据包）
    c.dispatchRequest(ch, p)
    
    // 3. 等待响应（支持超时取消）
    select {
    case <-ctx.Done():
        return 0, nil, ctx.Err()  // 上下文取消
    case s := <-ch:
        return s.typ, s.data, s.err  // 收到响应
    }
}

//dispatchRequest：请求分发机制
func (c *clientConn) dispatchRequest(ch chan<- result, p idmarshaler) {
    sid := p.id()  // 获取请求ID
    
    // 1. 将请求注册到inflight映射（关键步骤）
    if !c.putChannel(ch, sid) {
        return  // 连接已关闭
    }
    
    // 2. 发送数据包
    if err := c.conn.sendPacket(p); err != nil {
        // 发送失败时，立即通知调用方
        if ch, ok := c.getChannel(sid); ok {
            ch <- result{err: err}
        }
    }
}

//putChannel：请求注册阶段
func (c *clientConn) putChannel(ch chan<- result, sid uint32) bool {
    c.Lock()
    defer c.Unlock()
    
    // 检查连接状态
    select {
    case <-c.closed:
        ch <- result{err: ErrSSHFxConnectionLost}  // 连接已关闭
        return false
    default:
    }
    
    // 注册请求ID到结果通道的映射
    c.inflight[sid] = ch
    return true
}
```

上面代码中，通过`dispatchRequest`方法中`c.conn.sendPacket(p)`成功发送请求，在`recv`方法中获取response，并通过channel通知`sendPacket`方法的后续流程：

```GO
func (c *clientConn) recv() error {
    for {
        // 1. 接收原始数据包
        typ, data, err := c.recvPacket(0)
        if err != nil {
            return err
        }
        
        // 2. 解析请求ID（响应包的前4字节）
        sid, _, err := unmarshalUint32Safe(data)
        if err != nil {
            return err
        }
        
        // 3. 查找对应的等待通道（并清理）
        ch, ok := c.getChannel(sid)
        if !ok {
            return fmt.Errorf("sid not found: %d", sid)  // bad error：ID不匹配
        }
        
        // 4. 发送结果到对应的goroutine
        ch <- result{typ: typ, data: data}
    }
}

func (c *clientConn) getChannel(sid uint32) (chan<- result, bool) {
    c.Lock()
    defer c.Unlock()
    
    // 查找并立即删除（一次性使用）
    ch, ok := c.inflight[sid]
    delete(c.inflight, sid)  // 关键：防止内存泄漏
    
    return ch, ok
}
```

####	并发安全
注意到，上面的数据结构中`clientConn`的成员都是被锁保护的，主要是为了并发模式的安全性

```GO
type clientConn struct {
    sync.Mutex                    // 保护inflight映射的访问
    inflight map[uint32]chan<- result
    // - conn有自己的互斥锁保护发送操作
    // - closed通道的关闭操作是并发安全的
    // - err字段只在broadcastErr中修改（已持锁）
}
```

这里的并发指的是针对同一个sftp客户端，可以并发支持多个不同类型的操作：

```GO
// 同时发起多个文件操作
func main() {
    client, _ := sftp.NewClient(sshClient)
    
    var wg sync.WaitGroup
    
    // 并发读取多个文件
    files := []string{"file1.txt", "file2.txt", "file3.txt"}
    for i, filename := range files {
        wg.Add(1)
        go func(idx int, name string) {
            defer wg.Done()
            
            // 每个goroutine独立发送SFTP请求
            data, err := client.ReadFile(name)
            if err != nil {
                return
            }
			//done
        }(i, filename)
    }
    
    wg.Wait()
}
```

从sftp 客户端的主要操作[实现](https://github.com/pkg/sftp/blob/master/client.go)来看，每个原子操作的开始位置都会安全的生成一个全局id，用于并发处理

```GO
//https://github.com/pkg/sftp/blob/master/client.go#L737
func (c *Client) fstat(handle string) (*FileStat, error) {
	id := c.nextID()	//生成id
	typ, data, err := c.sendPacket(context.Background(), nil, &sshFxpFstatPacket{
		ID:     id,		//回带id
		Handle: handle,
	})
	if err != nil {
		return nil, err
	}
	//等待服务端响应
	switch typ {
	case sshFxpAttrs:
		sid, data := unmarshalUint32(data)
		if sid != id {
			//id校验
			return nil, &unexpectedIDErr{id, sid}
		}
		attr, _, err := unmarshalAttrs(data)
		return attr, err
	case sshFxpStatus:
		return nil, normaliseError(unmarshalStatus(id, data))
	default:
		return nil, unimplementedPacketErr(typ)
	}
}
```

####	sftp客户端支持的操作
以sftp客户端文件打开和文件读取为例，整体过程如下：

```TEXT
打开文件 → 获取文件句柄 → 分块读取 → 关闭文件
    |          |           |         |
SSH_FXP_OPEN -> SSH_FXP_READ -> ... -> SSH_FXP_CLOSE
```

通常的sftp客户端应用层代码如下：

```go
//打开服务器的文件
file, err := client.Open("/remote_path/file")
// 用户调用 Read 方法
buffer := make([]byte, 4096)

// 调用sftp file的Read方法
n, err := file.Read(buffer)
......
```

对应于sftp客户端的`Open`方法实现入口：

```GO
//https://github.com/pkg/sftp/blob/master/client.go#L654C1-L659C2
// Open opens the named file for reading. If successful, methods on the
// returned file can be used for reading; the associated file descriptor
// has mode O_RDONLY.
func (c *Client) Open(path string) (*File, error) {
	return c.open(path, toPflags(os.O_RDONLY))
}

func (c *Client) OpenFile(path string, f int) (*File, error) {
	return c.open(path, toPflags(f))
}

// open：客户端发送打开文件请求
func (c *Client) open(path string, pflags uint32) (*File, error) {
	// 生成唯一请求ID
	id := c.nextID()

	// 发送 SSH_FXP_OPEN 数据包
	typ, data, err := c.sendPacket(context.Background(), nil, &sshFxpOpenPacket{
		ID:     id,
		Path:   path,
		// 打开标志（只读、读写、创建等）
		Pflags: pflags,
	})
	if err != nil {
		return nil, err
	}

	//处理服务器响应
	switch typ {
	case sshFxpHandle:
		sid, data := unmarshalUint32(data)
		if sid != id {
			return nil, &unexpectedIDErr{id, sid}
		}
		handle, _ := unmarshalString(data)

		// 创建 File 对象，保存句柄和路径
		return &File{c: c, path: path, handle: handle/*服务器分配的文件句柄*/}, nil
	case sshFxpStatus:
		return nil, normaliseError(unmarshalStatus(id, data))
	default:
		return nil, unimplementedPacketErr(typ)
	}
}
```

对于上面的`Open`操作，相关的协议结构如下：

```GO
//https://github.com/pkg/sftp/blob/master/packet.go#L738
//客户端发送的 SSH_FXP_OPEN 包
type sshFxpOpenPacket struct {
	ID     uint32
	Path   string
	Pflags uint32
	Flags  uint32
	Attrs  interface{}
}

// 响应
type sshFxpHandlePacket struct {
	ID     uint32
	Handle string	 // 文件句柄（不透明字符串）
}
```

####	sftp文件读取过程
首先搞清楚一个概念，sftp文件读取`Read`方法，对应于下载文件场景：

```GO
func downloadFile(client *sftp.Client, remotePath, localPath string) error {
        // 注意：通过sftp打开远程文件
        remoteFile, err := client.Open(remotePath)
        if err != nil {
                return fmt.Errorf("打开远程文件失败: %w", err)
        }
        defer remoteFile.Close()

        // 创建本地目录（如果不存在）
        localDir := filepath.Dir(localPath)
        if err := os.MkdirAll(localDir, 0755); err != nil {
                return fmt.Errorf("创建本地目录失败: %w", err)
        }

        // 创建本地文件
        localFile, err := os.Create(localPath)
        if err != nil {
                return fmt.Errorf("创建本地文件失败: %w", err)
        }
        defer localFile.Close()

        // 复制文件内容
		// 注意：这里的remoteFile是sftp的File对象，会调用其实现的Read方法
        _, err = io.Copy(localFile, remoteFile)
        if err != nil {
                return fmt.Errorf("文件复制失败: %w", err)
        }

        // 确保数据写入磁盘
        return localFile.Sync()
}
```

`Client.Open`打开的远程对象就是`File`，其对应的`Read`实现意义为在远程服务器上读取数据，读取的数据存放在参数`b []byte`中

```GO
type File struct {
	c    *Client
	path string

	mu     sync.RWMutex
	handle string
	offset int64 // current offset within remote file
}

func (f *File) Read(b []byte) (int, error) {
	// 保护文件偏移量
	f.mu.Lock()	
	defer f.mu.Unlock()
	// 从当前偏移量开始读取
	n, err := f.readAt(b, f.offset)

	// 更新偏移量
	f.offset += int64(n)
	return n, err
}
```

`readAt`是整个sftp文件处理的核心实现，包含了三种模式：

1.	小文件`readChunkAt`
2.	顺序读取模式`readAtSequential`
3.	大文件

从这里实现也发现了一个细节，在 SFTP 协议中，`readAt` 机制中的大小选择对应的读取策略完全由 SFTP 客户端决定，服务端对此没有任何控制权，此外，客户端还控制了如下参数：

```GO
type Client struct {
	......
    maxPacket             int    // 客户端设置的最大包大小
    maxConcurrentRequests int    // 客户端设置的并发数限制
    disableConcurrentReads bool  // 客户端是否禁用并发读取
    useConcurrentWrites   bool   // 客户端是否使用并发写入
	......
}
```

继续看`readAt`的实现：

```GO
func (f *File) readAt(b []byte, off int64) (int, error) {
	if f.handle == "" {
		return 0, os.ErrClosed
	}

	if len(b) <= f.c.maxPacket {
		// This should be able to be serviced with 1/2 requests.
		// So, just do it directly.
		return f.readChunkAt(nil, b, off)
	}

	if f.c.disableConcurrentReads {
		return f.readAtSequential(b, off)
	}

	// Split the read into multiple maxPacket-sized concurrent reads bounded by maxConcurrentRequests.
	// This allows writes with a suitably large buffer to transfer data at a much faster rate
	// by overlapping round trip times.

	cancel := make(chan struct{})

	concurrency := len(b)/f.c.maxPacket + 1
	if concurrency > f.c.maxConcurrentRequests || concurrency < 1 {
		concurrency = f.c.maxConcurrentRequests
	}

	resPool := newResChanPool(concurrency)

	type work struct {
		id  uint32
		res chan result		//重要：接收响应（阻塞）

		b   []byte
		off int64
	}
	workCh := make(chan work)

	// Slice: cut up the Read into any number of buffers of length <= f.c.maxPacket, and at appropriate offsets.
	go func() {
		defer close(workCh)

		b := b
		offset := off
		chunkSize := f.c.maxPacket

		for len(b) > 0 {
			rb := b
			if len(rb) > chunkSize {
				rb = rb[:chunkSize]
			}

			id := f.c.nextID()
			res := resPool.Get()

			f.c.dispatchRequest(res, &sshFxpReadPacket{
				ID:     id,
				Handle: f.handle,
				Offset: uint64(offset),
				Len:    uint32(len(rb)),
			})

			select {
			case workCh <- work{id, res, rb, offset}:
			case <-cancel:
				return
			}

			offset += int64(len(rb))
			b = b[len(rb):]
		}
	}()

	type rErr struct {
		off int64
		err error
	}
	errCh := make(chan rErr)

	var wg sync.WaitGroup
	wg.Add(concurrency)
	for i := 0; i < concurrency; i++ {
		// Map_i: each worker gets work, and then performs the Read into its buffer from its respective offset.
		go func() {
			defer wg.Done()

			for packet := range workCh {
				var n int

				s := <-packet.res
				resPool.Put(packet.res)

				err := s.err
				if err == nil {
					switch s.typ {
					case sshFxpStatus:
						err = normaliseError(unmarshalStatus(packet.id, s.data))

					case sshFxpData:
						sid, data := unmarshalUint32(s.data)
						if packet.id != sid {
							err = &unexpectedIDErr{packet.id, sid}

						} else {
							l, data := unmarshalUint32(data)
							// 将 data[:l] copy到 packet.b
							n = copy(packet.b, data[:l])

							// For normal disk files, it is guaranteed that this will read
							// the specified number of bytes, or up to end of file.
							// This implies, if we have a short read, that means EOF.
							if n < len(packet.b) {
								err = io.EOF
							}
						}

					default:
						err = unimplementedPacketErr(s.typ)
					}
				}

				if err != nil {
					// return the offset as the start + how much we read before the error.
					errCh <- rErr{packet.off + int64(n), err}

					// DO NOT return.
					// We want to ensure that workCh is drained before wg.Wait returns.
				}
			}
		}()
	}

	// Wait for long tail, before closing results.
	go func() {
		wg.Wait()
		close(errCh)
	}()

	// Reduce: collect all the results into a relevant return: the earliest offset to return an error.
	firstErr := rErr{math.MaxInt64, nil}
	for rErr := range errCh {
		if rErr.off <= firstErr.off {
			firstErr = rErr
		}

		select {
		case <-cancel:
		default:
			// stop any more work from being distributed. (Just in case.)
			close(cancel)
		}
	}

	if firstErr.err != nil {
		// firstErr.err != nil if and only if firstErr.off > our starting offset.
		return int(firstErr.off - off), firstErr.err
	}

	// As per spec for io.ReaderAt, we return nil error if and only if we read everything.
	return len(b), nil
}
```

分三种情况对`readAt`进行分析：

1、小文件读取（客户端发送）

```GO
//https://github.com/pkg/sftp/blob/master/client.go#L1132
func (f *File) readChunkAt(ch chan result, b []byte, off int64) (n int, err error) {
	for err == nil && n < len(b) {
		id := f.c.nextID()
		// 发送 SSH_FXP_READ 请求
		typ, data, err := f.c.sendPacket(context.Background(), ch, &sshFxpReadPacket{
			ID:     id,
			Handle: f.handle,	 // 之前获取的文件句柄
			Offset: uint64(off) + uint64(n),	 // 读取偏移量
			Len:    uint32(len(b) - n),		 // 剩余要读取的长度
		})
		if err != nil {
			return n, err
		}

		switch typ {
		case sshFxpStatus:
			return n, normaliseError(unmarshalStatus(id, data))
		// 解析返回的数据
		case sshFxpData:
			sid, data := unmarshalUint32(data)
			if id != sid {
				return n, &unexpectedIDErr{id, sid}
			}
			// 提取数据长度和内容
			l, data := unmarshalUint32(data)
			// 复制到用户缓冲区
			n += copy(b[n:], data[:l])

		default:
			return n, unimplementedPacketErr(typ)
		}
	}

	return
}
```

2、`readAtSequential`，同上

```GO
func (f *File) readAtSequential(b []byte, off int64) (read int, err error) {
	for read < len(b) {
		rb := b[read:]
		if len(rb) > f.c.maxPacket {
			rb = rb[:f.c.maxPacket]
		}
		n, err := f.readChunkAt(nil, rb, off+int64(read))
		if n < 0 {
			panic("sftp.File: returned negative count from readChunkAt")
		}
		if n > 0 {
			read += n
		}
		if err != nil {
			return read, err
		}
	}
	return read, nil
}
```

3、大文件分块读取（大文件并发读取策略），适用于高性能处理的场景，这段代码：

-	核心功能是对大文件读取进行智能分片，然后并发发送多个小请求，最后组装响应（这是一个客户端的分块并发下载优化策略）
-	一个细节是远程文件的真正读取逻辑并不在客户端代码的实现中，而是在SFTP服务器端执行，再次强调客户端只负责拼装sftp请求
-	拆分请求（单个独立goroutine）、处理发送请求（多个独立的goroutine）

sftp客户端与服务端的基本分工如下（下载场景）

```TEXT
SFTP客户端                                          SFTP服务器端
-------------                 网络协议              -------------
1. 组织读取请求                SSH连接               1. 实际文件IO操作
2. 分块并发请求                SFTP协议              2. 读取磁盘文件  
3. 组装响应数据                                      3. 返回文件内容
4. 错误处理                                          4. 权限验证
```

```GO
// readAt：注意从服务端发回（下载）的数据最终存储在b中
func (f *File) readAt(b []byte, off int64) (int, error) {
	......
	cancel := make(chan struct{})

	// 计算合适的并发度（将b进行分块）
	concurrency := len(b)/f.c.maxPacket + 1
	if concurrency > f.c.maxConcurrentRequests || concurrency < 1 {
		concurrency = f.c.maxConcurrentRequests
	}

	resPool := newResChanPool(concurrency)

	// 创建工作池处理并发请求
	type work struct {
		id  uint32
		res chan result

		b   []byte
		off int64
	}
	workCh := make(chan work)

	// Slice: cut up the Read into any number of buffers of length <= f.c.maxPacket, and at appropriate offsets.
	go func() {
		defer close(workCh)

		b := b
		offset := off
		chunkSize := f.c.maxPacket

		// 分发工作任务（将大数据分块）
		// 将一个大读取分解为多个小读取请求
		for len(b) > 0 {
			rb := b
			if len(rb) > chunkSize {	// 通常 32KB
				rb = rb[:chunkSize]	// 切分为 32KB 的块
			}

			// 为每个块创建独立的读取请求
			id := f.c.nextID()	 // 生成唯一ID
			res := resPool.Get()

			// 重要：发送sftp请求
			f.c.dispatchRequest(res, &sshFxpReadPacket{
				ID:     id,
				Handle: f.handle,
				Offset: uint64(offset),		// 每个块的起始偏移
				Len:    uint32(len(rb)),	// 每个块的大小
			})

			select {
				//生产者：将大文件拆分（快）
			case workCh <- work{id, res, rb/*小块存储*/, offset}:
			case <-cancel:
				return
			}

			//继续下一个包（移动到下一个块）
			offset += int64(len(rb))
			b = b[len(rb):]
		} 
	}()

	type rErr struct {
		off int64
		err error
	}
	errCh := make(chan rErr)

	var wg sync.WaitGroup
	wg.Add(concurrency)
	// 启动worker goroutines并发处理（并发处理服务端响应+拼装数据）
	for i := 0; i < concurrency; i++ {
		// Map_i: each worker gets work, and then performs the Read into its buffer from its respective offset.
		go func() {
			defer wg.Done()

			//消费者：获取block基础元素
			for packet := range workCh {
				var n int

				// 重要：阻塞等待服务端响应结果
				s := <-packet.res
				resPool.Put(packet.res)

				err := s.err
				if err == nil {
					switch s.typ {
					case sshFxpStatus:
						err = normaliseError(unmarshalStatus(packet.id, s.data))

					case sshFxpData:
						sid, data := unmarshalUint32(s.data)
						if packet.id != sid {
							err = &unexpectedIDErr{packet.id, sid}

						} else {
							l, data := unmarshalUint32(data)

							// 重要：将服务端返回的数据data[:l] 复制到本地缓存区b中
							// 注意这个b是被分块的
							n = copy(packet.b, data[:l])

							// For normal disk files, it is guaranteed that this will read
							// the specified number of bytes, or up to end of file.
							// This implies, if we have a short read, that means EOF.
							if n < len(packet.b) {
								err = io.EOF
							}
						}

					default:
						err = unimplementedPacketErr(s.typ)
					}
				}

				if err != nil {
					// return the offset as the start + how much we read before the error.
					errCh <- rErr{packet.off + int64(n), err}

					// DO NOT return.
					// We want to ensure that workCh is drained before wg.Wait returns.
				}
			}
		}()
	}

	// Wait for long tail, before closing results.
	go func() {
		wg.Wait()
		close(errCh)
	}()

	// Reduce: collect all the results into a relevant return: the earliest offset to return an error.
	firstErr := rErr{math.MaxInt64, nil}
	for rErr := range errCh {
		if rErr.off <= firstErr.off {
			firstErr = rErr
		}

		select {
		case <-cancel:
		default:
			// stop any more work from being distributed. (Just in case.)
			close(cancel)
		}
	}

	if firstErr.err != nil {
		// firstErr.err != nil if and only if firstErr.off > our starting offset.
		return int(firstErr.off - off), firstErr.err
	}

	// As per spec for io.ReaderAt, we return nil error if and only if we read everything.
	return len(b), nil
}
```

## 0x06 参考
-	[ssh 包](https://pkg.go.dev/golang.org/x/crypto/ssh)
-	[gliderlabs-ssh 包](https://pkg.go.dev/github.com/gliderlabs/ssh)
-	[Understanding SSH](https://containerssh.io/development/containerssh/ssh/)
-	[SSH 协议 和 Go SSH 库源码浅析](https://www.rectcircle.cn/posts/ssh-protocol-and-go-lib/#channel)
-	[go 复用ssh 中的session_重新认识SSH（二）](https://blog.csdn.net/weixin_34887818/article/details/113074657)