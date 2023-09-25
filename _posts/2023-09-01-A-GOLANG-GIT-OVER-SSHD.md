---
layout: post
title: 使用 Golang 实现 SSH 和 SSHD（四）
subtitle: git over sshd：如何实现一个 git-sshd-server
date: 2023-09-01
author: pandaychen
catalog: true
tags:
  - OpenSSH
  - Git
---

## 0x00 git over ssh
项目中需要实现 git 客户端的命令审计等功能，本文介绍下 git over ssh(d) 的原理及实现细节。具体的细节可以参考项目：[gogs](https://github.com/gogs/gogs)，本文依此项目为基础介绍下实现细节；此外还可以参考：[git-Reference](https://git-scm.com/docs)

本文的关键词是：git over ssh authorized_keys with forced_command

####  git 支持的传输协议
git 目前主要支持的网络协议有如下三种：

- `http(s)://`
- `ssh://`
- `git://`

无论上述哪种协议，拉取实质上都是 `git-fetch-pack`/`git-upload-pack` 的数据交换，推送都是 `git-send-pack`/`git-receive-pack` 的数据交换。为简化，这里不介绍 Dump 哑协议。更详细的介绍可以参考：[传输协议](https://iissnan.com/progit/html/zh/ch9_6.html)

#### 1、HTTP(S) 传输（Smart 智能协议）

HTTP 智能协议与 [哑协议](https://xie.infoq.cn/article/ae4a65148cc85dd155011bead) 最大的区别在于：哑协议在获取数据时需自行指定文件资源的网络地址，并且通过多次下载操作来完成；而智能协议的则由服务端控制，服务端提供的 `info/refs` 可以动态更新，并且可以通过客户端传来的参数，决定本次交互客户端所需要的最小对象集，并打包压缩发给客户端，客户端会进行解压来拿到自己想要的数据。以 `git clone` 为例，整个交互过程如下（两次请求）：

![git-http](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/git/git-http-protocol.png)

1.	引用发现：`GET https://xxx.com/account/repo/info/refs?service=git-{upload|receive}-pack`
2.	数据传输：`POST https://xxx.com/account/repo/git-{upload|receive}-pack`

Git HTTP 协议要求操作前必须先执行引用发现（即需要知道服务端的各个引用的版本信息），这样的话才能让服务端或者客户端知道两方之间的差异以及需要什么样的数据。

1、引用发现 <br>
智能协议的的服务端是动态服务器，能够根据期望来提供相关的引用信息，通过抓包看到客户端请求的数据以及 git 服务端返回的引用信息格式，如下：

```TEXT
# 请求体
GET http://git.xxxx.net/pandaychen/getingblog.git/info/refs?service=git-upload-pack HTTP/1.1
Host: git.xxxx.net
User-Agent: git/2.24.3 (Apple Git-128)
Accept-Encoding: deflate, gzip
Proxy-Connection: Keep-Alive
Pragma: no-cache

# GIT server 响应
HTTP/1.1 200 OK
Cache-Control: no-cache, max-age=0, must-revalidate
Connection: keep-alive
Content-Type: application/x-git-upload-pack-advertisement
Expires: Fri, 01 Jan 1980 00:00:00 GMT
Pragma: no-cache
Server: nginx
X-Frame-Options: DENY
X-Git-Server: Brzox/3.2.3
X-Request-Id: 96e0af82-dffe-4352-9fa5-92f652ed39c7
Transfer-Encoding: chunked

001e# service=git-upload-pack
0000
010fca6ce400113082241c1f45daa513fabacc66a20d HEADmulti_ack thin-pack side-band side-band-64k ofs-delta shallow deepen-since deepen-not deepen-relative no-progress include-tag multi_ack_detailed no-done symref=HEAD:refs/heads/testbody object-format=sha1 agent=git/2.29.2
003c351bad7fdb498c9634442f0c3f60396e8b92f4fb refs/heads/dev
004092ad3c48e627782980f82b0a8b05a1a5221d8b74 refs/heads/dev-pro
0040ae747d0a0094af3d27ee86c33e645139728b2a9a refs/heads/develop
0000
```

上面的协议字段有部分值得注意的细节：


####  2、Git 传输协议
Git 协议以及 SSH 协议都是四层的传输协议，而 HTTP 则是七层的传输协议，受限于 HTTP 协议的特点，HTTP 在 Git 相关的操作上存在传输限制、超时等问题，这个问题在大仓库的传输中尤为明显，相较与 HTTP 协议，Git 以及 SSH 协议在传输上更稳定

![git-git](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/git/git-git-protocol.png)

Git 协议优势就是速度快（无加解密），但受限于协议的缺点，Git 协议常用于开源项目的下载，不作为私有项目的传输协议。Git 客户端跟服务端的交互主要包含两个步骤：

1.  获取服务端的引用
2.  客户端根据服务端的引用数据与服务端进行数据交换

Git 协议也遵循上述模式，只不过相比于 HTTP 协议，Git 协议直接在 TCP 层长连接直接完成上述 `2` 个步骤，如上图。在使用 Git 协议操作的时候，首先客户端会把相关的信息发给服务端，这个信息的格式同样的采用 `pkt-line` 的格式：

```TEXT
003egit-upload-pack /project.git\0host=myserver.com\0\0version=1\0
```

其中包含了命令、仓库名称、Host 等相关信息，服务端建立连接之后，接收到上述信息（协议数据），对其中的信息进行加工，找到对应的仓库的位置也就是目录，当所有的信息都符合要求之后，只需要在服务端启动 `upload-pack` 命令即可，这里需要注意的是我们不需要添加 `--stateless-rpc` 参数，直接 `git upload-pack {repo_path}`，这个命令启动后会马上返回相关的引用信息并且阻塞等待下一次信息的输入：

```bash
➜  hello git:(master) ✗ git upload-pack .
010234d8ed9a9f73d2cac9f50a8c8c03e4643990a2bf HEADmulti_ack thin-pack side-band side-band-64k ofs-delta shallow deepen-since deepen-not deepen-relative no-progress include-tag multi_ack_detailed symref=HEAD:refs/heads/master agent=git/2.24.3.(Apple.Git-128)
003f34d8ed9a9f73d2cac9f50a8c8c03e4643990a2bf refs/heads/master
0000
```

上述流程最终还是数据的转发，命令的标准输出信息需要原封不动的发送给客户端，客户端则会进行跟 HTTP 协议类似的处理产生数据，接着会把数据发给服务端，程序再原封不动的发给 `git upload-pack {repo_path}` 命令的标准输入，然后服务端处理完成后会把相应的包通过标准输出返回，程序再原封不动的发给客户端，就完成了一次 Fetch 操作，而 Push 的 `receive-pack` 操作原理相同；若客户端发送的信息不符合要求，或者处理过程中出现了问题，程序需要返回错误告知客户端，该错误的格式也是 `pkt-line`，以 `ERR` 开头，如下：

```GO
// error-line     =  PKT-LINE("ERR" SP explanation-text)
func exitSession(conn net.Conn, err error) {
	errStr := fmt.Sprintf("ERR %s", err.Error())
	pktData := fmt.Sprintf("%04x%s", len(errStr)+4, errStr)
	_, _ = conn.Write([]byte(pktData))
	_ = conn.Close()
}
```

客户端接收到上述错误信息后，就会打印信息并关闭连接

#### 3、SSH 传输协议
与 Git 协议比较，SSH 协议传输的数据需要加密。除此外，SSH 协议的传输过程与 Git 协议一致，都是跟服务端的进程做数据交换：

![git-ssh](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/git/git-ssh-protocol.png)

SSH 协议的下载地址一般是 `git@xxx.com:username/xxxx-repo.git` 形式，在执行 `clone` 或者 `push` 的时候，会拆解成下面指令：

```BASH
ssh username@xxx.com "git-upload-pack'/xxxx-repo.git'"
```

##	0x01  原理
Git over SSH 的原理是通过 SSH 协议来进行安全的远程访问和数据传输。git 借用了 ssh 的 Channel 中的 `exec` Request 类型，通过封装自己的核心 `git` 指令，来完成客户端与服务端的 [git 指令](https://github.com/pandaychen/gogs/blob/main/internal/cmd/serv.go#L129) 交互，重点是下面 `3` 个：

- `git-upload-pack`：https://git-scm.com/docs/git-upload-pack
- `git-upload-archive`
- `git-receive-pack`

上面这几个是 git 的低级命令（不建议使用），这几个命令封装了 git 工具的权限处理，内部 api 调用，git 上传 / 下载等操作（** 核心 **）

可以回想，一般 git ssh 支持需要客户端配置 ssh-keys 来辅助身份验证，原理是 git 利用了 ssh 的 `authorized_keys` 配合 `forced_command` 实现机制：
1.  利用 `authorized_keys` 来验证 git 客户端身份
2.  建立 ssh 连接后，使用 `forced_command` 机制来限制客户端仅仅调用 git 限制的指令
3.  操作完成后，ssh 连接退出
``
通常上面的 `forced_command`，在 git 中也称为 `git-shell`

####  Git with OpenSSH
在 OpenSSH 的 `sshd_config` 中有一些关键配置项 `AuthorizedKeysFile` 和 `AuthorizedKeysCommand`。Git SSH 的鉴权 / 执行一般过程如下：

```text
git pull/push over ssh -> gitlab-shell -> API call to gitlab-rails (Authorization) -> accept or decline -> execute git command
```

##  0x02  常用实现的 git-shell
列举下几款开源 git server 实现中的 `forced_command` 格式，注意，这个 `command` 需要满足如下几点：
1.  实现对应的 git 指令功能
2.  对接 ssh 客户端的 input/output

####  gitlab

```bash
command="/opt/gitlab/embedded/service/gitlab-shell/bin/gitlab-shell key-<db key id>",no-port-forwarding,no-X11-forwarding,no-agent-forwarding,no-pty <ssh public key>
```

####  gogs

```BASH
_TPL_PUBLICK_KEY = `command="%s serv key-%d --config='%s'",no-port-forwarding,no-X11-forwarding,no-agent-forwarding,no-pty %s` +"\n"\
```

上述 command 本质上是 gops 实现的 `serv` 指令：

![serv](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/git/gogs/gogs_client_serv_1.png)

#### [codefever](https://github.com/PGYER/codefever/tree/master)

```BASH
command="PATH=$PATH:/usr/local/git/bin && /data/www/codefever-community/ssh-gateway/shell/main $SSH_ORIGINAL_COMMAND 00000000000000000000000000000000",no-port-forwarding,no-X11-forwarding,no-agent-forwarding,no-pty <ssh public key>
```

##  0x03  git over ssh：前置知识
本小节介绍 Git over SSH Server 技术细节。Git SSH Server 与 SSH Server 的主要区别是，Git SSH Server 只是一个严格的子集，session 阶段只需要实现 `env` 和 `exec` 两种请求即可，不需要完整实现如 `shell`、`pty-req` 等请求。

##  0x03  gogs ：git over sshd 分析


##  0x0 补充：git over HTTP


##  0x0 总结
关于 git 的使用 / 经验 / 原理或者更多细节，可以延伸阅读如下文章：
- [代码托管从业者 Git 指南](https://ipvb.gitee.io/git/2021/01/21/GitGuideForCodeHostingPractitioners/)

##  0x04 参考
- [GOGS：Gogs is a painless self-hosted Git service](https://github.com/gogs/gogs)
- [几款开源 git server ssh 协议 forced command 参考格式](https://www.cnblogs.com/rongfengliang/p/15924984.html)
- [10.6 Git 内部原理 - 传输协议](https://git-scm.com/book/zh/v2/Git-%E5%86%85%E9%83%A8%E5%8E%9F%E7%90%86-%E4%BC%A0%E8%BE%93%E5%8D%8F%E8%AE%AE)
- [SSH access for GitLab](https://gitlab.com/gitlab-org/gitlab-shell)
- [聊聊 Git 的三种传输协议及实现](https://xie.infoq.cn/article/ae4a65148cc85dd155011bead)
- [构建恰当的 Git SSH Server](https://forcemz.net/git/2019/03/16/MakeAGitSSHServer/)
- [代码托管从业者 Git 指南](https://ipvb.gitee.io/git/2021/01/21/GitGuideForCodeHostingPractitioners/)
-	[传输协议](https://iissnan.com/progit/html/zh/ch9_6.html)