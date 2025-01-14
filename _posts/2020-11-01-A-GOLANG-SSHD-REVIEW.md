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

##	0x04	subsystem
如何理解 sshd 中的 subsytem 机制？subsystem 是 SSH 协议的一个扩展功能，它允许 SSH 服务器在远程客户端的请求下，启动特定的预定义服务。这些服务可以是文件传输、版本控制系统、远程桌面等。subsytem 是一种 ** 在 SSH 会话中运行特定应用程序的方法，而无需在远程计算机上启动一个完整的 shell 会话 **。`sftp-server` 就是典型的 subsytem 服务，在 `/etc/ssh/sshd_config` 一般可以看到 `Subsystem sftp /usr/libexec/openssh/sftp-server` 的配置选项，当客户端发起一个 SFTP 会话时，sshd 将启动 `/usr/lib/openssh/sftp-server` 程序，并将其与客户端的 SSH 会话关联。

此外，还有几种典型的场景：
- git server：`Subsystem git /usr/bin/git-shell`
- x11 转发：`Subsystem x11 /usr/bin/Xvnc`

子系统的优势是可以为特定任务提供专用的环境。如使用 SFTP 子系统，用户可以在不启动一个完整的远程 shell 会话的情况下，安全地传输文件。这有助于提高安全性，同时减少了系统资源的消耗

#### gliderlabs/ssh 的子系统
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

最后再看下 `github.com/pkg/sftp` 的启动 [相关代码](https://github.com/pkg/sftp/blob/master/server.go#L351)`：

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
	runWorker := func(ch chan orderedRequest) {
		wg.Add(1)
		go func() {
			defer wg.Done()
			if err := svr.sftpServerWorker(ch); err != nil {
				svr.conn.Close() // shuts down recvPacket
			}
		}()
	}
	pktChan := svr.pktMgr.workerChan(runWorker)

	var err error
	var pkt requestPacket
	var pktType uint8
	var pktBytes []byte
	for {
		// 从 ssh 会话接受数据
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

#### 客户端调用子系统


## 0x05 参考
-	[ssh 包](https://pkg.go.dev/golang.org/x/crypto/ssh)
-	[gliderlabs-ssh 包](https://pkg.go.dev/github.com/gliderlabs/ssh)
-	[Understanding SSH](https://containerssh.io/development/containerssh/ssh/)
-	[SSH 协议 和 Go SSH 库源码浅析](https://www.rectcircle.cn/posts/ssh-protocol-and-go-lib/#channel)
-	[go 复用ssh 中的session_重新认识SSH（二）](https://blog.csdn.net/weixin_34887818/article/details/113074657)