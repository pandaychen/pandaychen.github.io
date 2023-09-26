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

无论上述哪种协议，拉取实质上都是 `git-fetch-pack`/`git-upload-pack` 的数据交换，推送都是 `git-send-pack`/`git-receive-pack` 的数据交换。更详细的介绍可以参考：[传输协议](https://iissnan.com/progit/html/zh/ch9_6.html)

#### 1、HTTP（Dump 哑协议）
Git 哑协议仅需要一个标准的 HTTP 静态文件服务（能够提供文件的下载即可），Git 客户端会自动进行文件的遍历和拉取。无论是哑协议还是智能协议，Git 在使用 HTTP 协议进行 `Fetch` 操作的时候，总是要先获取 `info/refs` 文件（地址为 `.git/info/refs`，可通过 `git update-server-info` 生成）；可以配置 Git 服务端的 `post-receive` 钩子自动执行 `update-server-info` 更新

```BASH
[root@VM_120_245_centos /tmp/gogs/.git]# cat ./info/refs
018337ddfbd33495a08ece1bc4ab639aba730142        refs/heads/main
018337ddfbd33495a08ece1bc4ab639aba730142        refs/remotes/origin/HEAD
4154f528c3d3c0d60e1919bda9eed6a500e49e81        refs/remotes/origin/jc/db-migrate-orgs
b9266247a4701c20c0297e4a8c965bac9b888365        refs/remotes/origin/jc/exp/pack-release-archives-in-docker
f1a4b8683b2338b198114786a0f4cba14e8d07e8        refs/remotes/origin/jc/exp/srcgraph-external-service
018337ddfbd33495a08ece1bc4ab639aba730142        refs/remotes/origin/main
c9fba3cb30af0789fcf89098dfcb8f2286ee7d3b        refs/remotes/origin/release/0.12
8c21874c00b6100d46b662f65baeb40647442f42        refs/remotes/origin/release/0.13
b3757e424ffc47f7ae07d8fecd9f2ecf98f20679        refs/tags/v0.10
9d40b8a83cc3a13ec0859ad253de02c514e4403d        refs/tags/v0.10.1
f54bcba3394bf856b77b674203ea0c80b926cd61        refs/tags/v0.10.18
bb005f3f9a606a5e94da4fc274d3c21234d98090        refs/tags/v0.10.8
.......
```

文件内容主要是服务端上每个引用的版本，客户端拿到这些引用之后，就可以跟本地的引用进行对比，对于缺失的对象文件，则通过 HTTP 的方式进行下载。一次通过哑协议 `Clone` 的过程如下：

1.	用户：`git clone https://xxx.com/pandaychen/abc.git`
2.	客户端：`GET https://xxx.com/pandaychen/abc.git/info/refs`
3.	服务端：Response with `abc.git/info/refs`
4.	客户端：`GET https://xxx.com/pandaychen/abc.git/HEAD` （默认分支）
5.	服务端：Response with `abc.git/HEAD`
6.	客户端：Get `https://xxx.com/pandaychen/abc.git/objects/ef/8021acf4c29eb35e3084b7dc543c173d67ad2a` 开始遍历对象，找出那些本地没有的，去服务端获取，如果服务端无法直接获取，则从 Pack 文件中进行抓取，直到全部拿到
7.	客户端：根据 `HEAD` 中的默认分支执行 `checkout` 操作检出到本地

可以通过 `http.FileServer` 直接构建一个 dump server（也可以直接以 nginx 文件服务器提供）：

```GO
func main() {
	repo := flag.String("repo", "/root/repositories", "Specify a repositories root dir.")
	port := flag.String("port", "8888", "Specify a port to start process.")
	flag.Parse()

	http.Handle("/", http.FileServer(http.Dir(*repo)))
	fmt.Printf("Dumb http server start at port %s on dir %s \n", *port, *repo)
	_ = http.ListenAndServe(fmt.Sprintf(":%s", *port), nil)
}
```

#### 2、HTTP(S) 传输（Smart 智能协议）

HTTP 智能协议与 [哑协议](https://xie.infoq.cn/article/ae4a65148cc85dd155011bead) 最大的区别在于：哑协议在获取数据时需自行指定文件资源的网络地址，并且通过多次下载操作来完成；而智能协议的则由服务端控制，服务端提供的 `info/refs` 可以动态更新，并且可以通过客户端传来的参数，决定本次交互客户端所需要的最小对象集，并打包压缩发给客户端，客户端会进行解压来拿到自己想要的数据。以 `git clone` 为例，整个交互过程如下（两次请求）：

![git-http](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/git/git-http-protocol.png)

1.	引用发现：`GET https://xxx.com/account/repo/info/refs?service=git-{upload|receive}-pack`
2.	数据传输：`POST https://xxx.com/account/repo/git-{upload|receive}-pack`

Git HTTP 协议要求操作前必须先执行引用发现（即需要知道服务端的各个引用的版本信息），这样的话才能让服务端或者客户端知道两方之间的差异以及需要什么样的数据。

1、引用发现 <br>
智能协议的的服务端是动态服务器，能够根据期望来提供相关的引用信息，通过抓包看到客户端请求的数据以及 git 服务端返回的引用信息格式，如下：

```TEXT
# 请求体
GET http://git.xxxx.net/pandaychen/abc.git/info/refs?service=git-upload-pack HTTP/1.1
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

上面的协议字段及响应有部分值得注意的细节：

-	`Cache-Control`：必须禁止缓存，不然可能看不到最新的提交
-	`Content-Type`：必须是 `application/x-$servicename-advertisement`，不然客户端会以哑协议的方式去处理
-	客户端需要验证返回的状态码，如果是 `401` 那么就提示输入用户名密码
-	智能协议响应 Response Body 格式跟哑协议所用的 `info/refs` 内容不一样，客户端需要根据这个来识别支持的属性和验证信息（`pkt-line` 格式）

`pkt-line` 协议格式定义如下：

1.	客户端需要验证第一首行的 `4` 个字符符合正则 `^[0-9a-f]{4}#`，这里的四个字符是代表后面内容的长度
2.	客户端需要验证第一行是# `service=$servicename`
3.	服务端得保证每一行结尾需要包含一个 `LF` 换行符
4.	服务端需要以 `0000` 标识结束本次请求响应

可以通过 git 命令行 `git upload-pack --stateless-rpc --advertise-refs .` 获取某个 repo 下面的 `info/refs` 的 `pkt-line` 格式 `：

```txt
[root@VM_120_245_centos /tmp/gogs]# git upload-pack --stateless-rpc --advertise-refs .
00b5018337ddfbd33495a08ece1bc4ab639aba730142 HEADmulti_ack thin-pack side-band side-band-64k ofs-delta shallow no-progress include-tag multi_ack_detailed no-done agent=git/1.8.3.1
003d018337ddfbd33495a08ece1bc4ab639aba730142 refs/heads/main
0046018337ddfbd33495a08ece1bc4ab639aba730142 refs/remotes/origin/HEAD
00544154f528c3d3c0d60e1919bda9eed6a500e49e81 refs/remotes/origin/jc/db-migrate-orgs
0068b9266247a4701c20c0297e4a8c965bac9b888365 refs/remotes/origin/jc/exp/pack-release-archives-in-docker
0062f1a4b8683b2338b198114786a0f4cba14e8d07e8 refs/remotes/origin/jc/exp/srcgraph-external-service
0046018337ddfbd33495a08ece1bc4ab639aba730142 refs/remotes/origin/main
004ec9fba3cb30af0789fcf89098dfcb8f2286ee7d3b refs/remotes/origin/release/0.12
004e8c21874c00b6100d46b662f65baeb40647442f42 refs/remotes/origin/release/0.13
003db3757e424ffc47f7ae07d8fecd9f2ecf98f20679 refs/tags/v0.10
......
0000 vv
```

`upload-pack` 是用来发送对象给客户端的一个远程调用模块，通过此指令，能够快速拿到当前的引用状态并退出，需要在服务端的裸仓库目录执行就可以直接拿到最新的引用信息，模拟实现的代码如下：

```GO
// 支持 upload-pack 和 receive-pack 两种操作引用发现的处理
func handleRefs(w http.ResponseWriter, r *http.Request) {
	vars := mux.Vars(r)
	repoName := vars["repo"]
	repoPath := fmt.Sprintf("%s%s", *repoRoot, repoName)
	service := r.FormValue("service")
	pFirst := fmt.Sprintf("# service=%s\n", service) // 本示例仅处理 protocol v1

	handleRefsHeader(&amp;w, service) // Headers 处理

	//call git command
	cmdRefs := exec.Command("git", service[4:], "--stateless-rpc", "--advertise-refs", repoPath)
	refsBytes, _ := cmdRefs.Output() // 获取 pkt-line 数据
	responseBody := fmt.Sprintf("%04x# service=%s\n0000%s", len(pFirst)+4, service, string(refsBytes)) // 拼接 Body

	_, _ = w.Write([]byte(responseBody))
}

// 按要求设置 Headers
func handleRefsHeader(w *http.ResponseWriter, service string) {
	cType := fmt.Sprintf("application/x-%s-advertisement", service)
	(*w).Header().Add("Content-Type", cType)
	(*w).Header().Set("Expires", "Fri, 01 Jan 1980 00:00:00 GMT")
	(*w).Header().Set("Pragma", "no-cache")
	(*w).Header().Set("Cache-Control", "no-cache, max-age=0, must-revalidate")
}
```

2、数据传输 <br>

-	客户端向服务端传输（Push）：Push 操作获取到服务端的引用列表后，由客户端本地计算出客户端所缺失的数据，将这些数据打包，并 `POST` 给服务端，服务端接收到后进行解压和引用更新
-	服务端向客户端传输（Fetch）：Fetch 操作在获取引用发现之后，由服务端计算出客户端想要的数据，并把数据以 `pkt-line` 格式 `POST` 给服务端，由服务端进行 Pack 的计算和打包，将包作为 `POST` 的响应发送给客户端，客户端进行解压和引用更新；Fetch 操作用到了 `upload-pack`，该指令是一个发送对象给客户端的远程调用模块，只需要在服务端启动 `git upload-pack --stateless-rpc` ，该命令阻塞的接收一串参数，而这串参数是客户端的第二次请求发送过来的，把它传递给这个命令，Git 就会自动的计算客户端所需要的最小对象集并打包，以流的形式返回这个包数据，最后只需要把这个包作为 `POST` 请求的响应发给客户端即可


在 Fetch 操作中，客户端第二次 `POST` 请求发过来的数据格式如下：

```txt
POST http://xxx.com/pandaychen/abc/git-upload-pack HTTP/1.1
Host: xxx.com
User-Agent: git/2.24.3 (Apple Git-128)
Accept-Encoding: deflate, gzip
Proxy-Connection: Keep-Alive
Content-Type: application/x-git-upload-pack-request
Accept: application/x-git-upload-pack-result
Content-Length: 443

00b4want bee4d57e3adaddf355315edf2c046db33aa299e8 multi_ack_detailed no-done side-band-64k thin-pack include-tag ofs-delta deepen-since deepen-not agent=git/2.24.3.(Apple.Git-128)
00000032have 82a8768e7fd48f76772628d5a55475c51ea4fa2f
0032have 4f7a2ea0920751a5501debbbc1debc403b46d7a0
0032have 7c141974a30bd218d4257d4292890a9008d30111
0032have f6bb00364bd5000a45185b9b16c028f485e842db
0032have 47b7bd17fcb7de646cf49a26efb43c7b708498f3
0009done
```

整个数据传输的过程无非就是客户端与服务端的 `upload-pack` 和 `receive-pack` 对规定格式的数据交换而已，第二步的处理的参考代码如下：

```GO
func processPack(w http.ResponseWriter, r *http.Request) {
	vars := mux.Vars(r)
	repoName := vars["repo"]
	// request repo not end with .git is supported with upload-pack
	repoPath := fmt.Sprintf("%s%s", *repoRoot, repoName)
	service := vars["service"]

	handlePackHeader(&amp;w, service)

	// 启动一个进程，通过标准输入输出进行数据交换
	cmdPack := exec.Command("git", service[4:], "--stateless-rpc", repoPath)
	cmdStdin, err := cmdPack.StdinPipe()
	cmdStdout, err := cmdPack.StdoutPipe()
	_ = cmdPack.Start()

	// 客户端和服务端数据交换
	go func() {
		_, _ = io.Copy(cmdStdin, r.Body)
		_ = cmdStdin.Close()
	}()
	_, _ = io.Copy(w, cmdStdout)
	_ = cmdPack.Wait() // wait for std complete
}
```

####  3、Git 传输协议
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

#### 4、SSH 传输协议
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


##	0x04	gitlab shell 原理
参考 [GitLab Shell development guidelines](https://docs.gitlab.com/ee/development/gitlab_shell/)

When you access the GitLab server over SSH, GitLab Shell then:

1.	Limits you to predefined Git commands (git push, git pull, git fetch)
2.	Calls the GitLab Rails API to check if you are authorized, and what Gitaly server your repository is on
3.	Copies data back and forth between the SSH client and the Gitaly server

####	git pull over SSH
![gitlab-pull]()

####	git push over SSH
![gitlab-push]()

####	 flow of gitlab-shell
GitLab SSH 守护程序（gitlab-sshd）的运行流程如下：

1.	用户通过 SSH 协议连接到 gitlab-sshd
2.	gitlab-sshd 会验证用户的身份，并将用户的请求转发到 GitLab Rails 应用程序
3.	GitLab Rails 应用程序会验证用户的权限，并根据请求的类型执行相应的操作。例如，如果用户请求克隆存储库，则 GitLab Rails 应用程序会调用 Gitaly，将存储库的数据发送回 gitlab-sshd，然后将其转发给用户
4.	gitlab-sshd 将从 GitLab Rails 应用程序收到的响应发送回用户

在整个过程中，gitlab-sshd 充当了中间人的角色，负责将用户请求转发到 GitLab Rails 应用程序，并将响应发送回用户。同时，gitlab-sshd 还负责管理 SSH 连接，包括验证用户身份和维护连接状态

![gitlab-flow]()

在上图中，包含了两个中间件：

-	Rails：Rails 是一种 Web 应用程序框架，它为 GitLab 提供了一个基础架构，包括用户身份验证、API、Web 界面等。Rails 还提供了一些功能，例如存储库浏览器、问题跟踪器、CI/CD 等
-	Gitaly：Gitaly 是 GitLab 的分布式版本控制系统，它负责处理 Git 存储库的所有操作。Gitaly 是在 GitLab 中实现高可用性和可扩展性的关键组件。它将 Git 操作转换为 gRPC 调用，并将它们路由到 Git 存储库的正确实例。Gitaly 还提供了许多其他功能，例如存储库克隆、提交、拉取请求等

####	GitLab Shell architecture

![gitlab-arch]()

##  0x04 补充：git over HTTP


##  0x05 总结
关于 git 的使用 / 经验 / 原理或者更多细节，可以延伸阅读如下文章：
- [代码托管从业者 Git 指南](https://ipvb.gitee.io/git/2021/01/21/GitGuideForCodeHostingPractitioners/)

##  0x06 参考
-	[git 使用与原理剖析及其私服搭建](https://xie.infoq.cn/article/38e92927b2ca83f0d29224e6e)
- [GOGS：Gogs is a painless self-hosted Git service](https://github.com/gogs/gogs)
- [几款开源 git server ssh 协议 forced command 参考格式](https://www.cnblogs.com/rongfengliang/p/15924984.html)
- [10.6 Git 内部原理 - 传输协议](https://git-scm.com/book/zh/v2/Git-%E5%86%85%E9%83%A8%E5%8E%9F%E7%90%86-%E4%BC%A0%E8%BE%93%E5%8D%8F%E8%AE%AE)
- [SSH access for GitLab](https://gitlab.com/gitlab-org/gitlab-shell)
- [聊聊 Git 的三种传输协议及实现](https://xie.infoq.cn/article/ae4a65148cc85dd155011bead)
- [构建恰当的 Git SSH Server](https://forcemz.net/git/2019/03/16/MakeAGitSSHServer/)
- [代码托管从业者 Git 指南](https://ipvb.gitee.io/git/2021/01/21/GitGuideForCodeHostingPractitioners/)
-	[传输协议](https://iissnan.com/progit/html/zh/ch9_6.html)
-	[一个 golang 模拟 git 服务端的实现](https://gitee.com/kesin/go-git-protocols/tree/master/http-smart)
-	[Writing an SSH server in Go](https://blog.gopheracademy.com/advent-2015/ssh-server-in-go/)
-	[自己开发 git 服务器，有哪些开源可以借用？](https://www.zhihu.com/question/283143882)
-	[GitLab Shell development guidelines](https://docs.gitlab.com/ee/development/gitlab_shell/)