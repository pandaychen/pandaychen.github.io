---
layout:     post
title:      一种基于 TTY-based 的 kubernetes console 实现思路
subtitle:   如何使用 ssh 协议打通 kubernetes EXEC 登录？
date:       2022-06-01
author:     pandaychen
catalog:    true
tags:
    - Kubernetes
---

##  0x00    开篇
网上基于 [websocket](https://github.com/gorilla/websocket) 打通 kubernetes pod 的实现非常多，但是受限于 webconsole 的不便利性，这边文章来分析下如何使用 SSH 方式打通 kubernetes 的登录


####	Kubectl exec 的原理简介
通过 `kubectl exec` 进行容器的数据流如下：

![kubernetes-exec](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/kubernetes/kubectl-principle3.png)



##  0x01    实现思路

1、remotecommand 暴露的 `Executor` 方法 <br>
实现的核心是打通 IO（即 `stdin` 与 `stdout`）之间的桥接，如下图：

![flow](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/ssh/ssh2kubernetesflow.png)

kubernetes 的 [client-go](https://github.com/kubernetes/client-go/blob/master/tools/remotecommand/remotecommand.go) 包提供了集群中的容器建立长连接的方法，并设置容器的 `stdin`，`stdout` 等。此 package 提供了基于 SPDY 协议的 `Executor`，其本质是一个 `interface`，用于和 pod 终端的流的传输及通信。`Executor` 接口定义了 `Stream` [方法](https://github.com/kubernetes/client-go/blob/master/tools/remotecommand/remotecommand.go#L108)，该方法会建立一个流传输的连接，直到服务端和调用端一端关闭连接，才会停止传输（原 package 中给出的实现是基于 HTTP-SPDY 的，不适用于笔者的场景，所有需要重新实现 `Stream` 方法）

此外，考虑到 `StreamOptions` 是 `Stream` 方法的参数，在实现 `Stream` 方法时需要填充该结构。

```golang
// Executor is an interface for transporting shell-style streams.
type Executor interface {
	// Stream initiates the transport of the standard shell streams. It will transport any
	// non-nil stream to a remote system, and return an error if a problem occurs. If tty
	// is set, the stderr stream is not used (raw TTY manages stdout and stderr over the
	// stdout stream).
	Stream(options StreamOptions) error
}

func (e *streamExecutor) Stream(options StreamOptions) error{}

// StreamOptions holds information pertaining to the current streaming session:
// input/output streams, if the client is requesting a TTY, and a terminal size queue to
// support terminal resizing.
type StreamOptions struct {
	Stdin             io.Reader
	Stdout            io.Writer
	Stderr            io.Writer
	Tty               bool
	TerminalSizeQueue TerminalSizeQueue
}
```

关于 `SteamOption` 中各个成员的作用，可以参见 [V1](https://github.com/kubernetes/client-go/blob/master/tools/remotecommand/v1.go) 的实现；简言之，就是外部对接容器内部暴露的输入 / 输出。

####	TerminalSizeQueue 结构
TerminalSizeQueue is capable of returning terminal resize events as they occur：`TerminalSizeQueue` 结构的作用是向容器进行时传递客户端侧的窗口变化的通知事件（事件包含当前的终端的 `Weight` 与 `Height` 值）：即当客户终端 TTY 窗口的 `size` 发生了改变时，需要通知容器进行时调整显示的 `size`，否则会出现内外窗口显示不一致的问题

```golang
type TerminalSizeQueue interface {
	// Next returns the new terminal size after the terminal has been resized. It returns nil when
	// monitoring has been stopped.
	Next() *TerminalSize
}
```

[TerminalSize](https://github.com/kubernetes/client-go/blob/v0.24.4/tools/remotecommand/resize.go#L29) 结构如下：
```golang
// TerminalSize and TerminalSizeQueue was a part of k8s.io/kubernetes/pkg/util/term
// and were moved in order to decouple client from other term dependencies

// TerminalSize represents the width and height of a terminal.
type TerminalSize struct {
	Width  uint16
	Height uint16
}
```


2、定义核心结构体，构造 `StreamOptions` 所需的成员 <br>
```golang
// TtyHandler
type PyStreamOption interface {
	io.Reader	//stdin
	io.Writer	//stdout && stderr
	//io.Writer
	remotecommand.TerminalSizeQueue
}
```
定义如上结构 `PyStreamOption`，然后需要实例化 `PyStreamOption`；注意实例化之后，必须实现该 `interface` 的 `3` 个方法：

-	`Read(p []byte) (int, error)`：从前置 SSH 会话（ssh.Channel）中读取数据，转发到用户侧
-	`Write(p []byte) (int, error)`：从前置 SSH 会话获取用户输入，写入 pod
-	`Next() *TerminalSize`：即时调整终端的 size

设置的目的是：
1.	对接 terminal 的输入输出
2.	实时监听窗口大小的改变并调整

这样，后续调用 `Stream` 方法时，只要将 `StreamOptions` 的 `Stdin`/`Stdout` 都设置为 `PyStreamOption`，`Executor` 会通过你定义的 `Write` 和 `Read` 方法来传输数据。

3、建立容器（连接）的方法 <br>
注意下面的 `CreateK8SStream` 方法，最后会使用 `remotecommand` 的 `exec.Stream` 方法来实现打通容器登录的过程（`stdin`/`stdout`/`stderr` 由参数传入），即上一步定义的 `TtyHandler` 结构的相关成员：
```golang
func CreateK8SStream(option PyStreamOption) {
	//...
	//NewSPDYExecutor
	req := cli.CoreV1().RESTClient().Post().
			Resource("pods").
			Name(podName).
			Namespace(namespace).
			SubResource("exec")
	option := &corev1.PodExecOptions{
		Command: cmd,
		Stdin:   true,
		Stdout:  true,
		Stderr:  true,
		TTY:     true,
	}
	req.VersionedParams(
		option,
		scheme.ParameterCodec,
	)
	exec, err := remotecommand.NewSPDYExecutor(config, "POST", req.URL())
	if err != nil {
		return err
	}

	//Stream
	err = exec.Stream(remotecommand.StreamOptions{
		Stdin:             stdin,		// 来自 option
		Stdout:            stdout,		// 来自 option
		Stderr:            stderr,		// 来自 option
		Tty:               true,		// 来自 option，设置 TTY
		TerminalSizeQueue: terminalSizeQueue,	// 来自 option
	})
	//...
}
```

接下来的思路就是如何生成一个 `Executor`，来打通 SSH 到 SPDY（主要还是打通上图的 STDIN/STDOUT）

##	0x02	文件 COPY 的实现
如前文描述，方法 `exec.Stream` 执行时会向 APServer 发起一个请求，APServer 会通过代理机制将请求转发给相应节点上的 kubelet 服务，kubelet 会通过 CRI 接口调用 runtime 的接口，最终调用流式接口中的 `Exec()` 进入到 container 执行。通常针对一般化的命令调用，输入和输出以文本为主，并不会占用带宽和 APServer 的资源，但是当涉及到文件传输的时候则会占用较多带宽，因此这种方式不太适合大容量文件的 copy，如何优化？

1.	增加流控
2.	在文件 copy 场景下，直接向 kubelet 发起 `exec` 请求（绕过 APServer），但是需要解决访问 kubelet 地址、认证等相关问题

以文件 copy 场景为例，使用的命令是 `kubectl cp`，**前提是 Pod 中对应的 container 中安装了 `tar` 命令，本质上该指令还是调用了 `exec` 接口，然后执行 `tar` 命令进行文件的拷贝动作**

```BASH
#宿主机 –> Pod
#将宿主机 /tmp/test_pod.txt copy 到 mycentos-7b59b5b755-8rbgc 对应的第一个容器中的 /root 目录下
kubectl cp /tmp/test_pod.txt default/mycentos-7b59b5b755-8rbgc:/root

#Pod –> 宿主机
#将 mycentos-7b59b5b755-8rbgc 中 /root/from_pod.txt 文件拷贝到宿主机 /tmp 目录下，并命名为 from_pod.new
kubectl cp default/mycentos-7b59b5b755-8rbgc:/root/from_pod.txt  /tmp/from_pod.new
#将 mycentos-7b59b5b755-8rbgc 中 /root/pod_dir 目录下的文件拷贝到宿主机 /tmp 目录下
kubectl cp default/mycentos-7b59b5b755-8rbgc:/root/pod_dir /tmp
```

为什么要使用 `tar` 呢？`tar` 支持压缩从标准输入读取文件名列表

```BASH
find . -type f -name "*.jpg" -print | xargs tar -czvf images.tar.gz
find $HOME -name "*.doc" -print0 | tar -cvf /tmp/file.tar --null -T -
find $HOME -type f -name "*.sh" | xargs tar cfvz /nfs/x230/my-shell-scripts.tgz

# 从容器向外拷贝，通过 tar 命令压缩将输出重定向到标准输出
tar cf - ${srcFile}
# 从外面向容器内拷贝，将标准输入数据通过 tar 解压写入到 container 内的文件目录
tar --no-same-permissions --no-same-owner -xmf -
# OR
tar -xmf -
```

通过 `client-go` 向 pod 所在节点实现查询 kubelet 地址，然后向 kubelet 直接发起 `exec` 请求，代码如下：

```go
func main() {
        // 此处直接使用了 / root/.kube/config 文件（有足够的权限）
        // 如果使用 service account token 的话，还需要额外创建 role 和 rolebinding
        // 当直接访问 kubelet 接口的时候，kubelet 支持使用证书和 token 进行认证，
        // 使用 rbac 对请求进行鉴权操作
        var kubeConfig *string
        if home := homedir.HomeDir(); home != "" {
                kubeConfig = flag.String("kubeconfig", filepath.Join(home, ".kube", "config"), "(optional) absolute path to the kubeconfig file")
        } else {
                kubeConfig = flag.String("kubeconfig", "","absolute path to the kubeconfig file")
        }
        flag.Parse()

        config, err := clientcmd.BuildConfigFromFlags("", *kubeConfig)
        if err != nil {
                panic(err)
        }

        // 构建请求，直接请求 kubelet 需要的各种参数
        params := url.Values{}
        params.Add("tty", "1")
        params.Add("input", "0")
        params.Add("output", "1")
        params.Add("error", "0")
        params.Add("command", "ls")
        params.Add("command", "/")
        url := &url.URL{
                Scheme:   "https",
                Host:     "10.0.0.1:10250",
                Path:     "/exec/default/busybox/busybox1",
                RawQuery: params.Encode(),
        }
        // 此处配置证书相关配置，由于是测试，这块直接忽略了，在实际生产使用时可从文件读取
        // 此处配置的 CAFile 在更新后，client-go 能够定时自动刷新
        config.TLSClientConfig.CAFile = ""
        config.TLSClientConfig.CAData = nil
        config.TLSClientConfig.Insecure = true
        executor, err := remotecommand.NewSPDYExecutor(config, "POST", url)
        if err != nil {
                panic(err)
        }

        done := make(chan error)
        var buf bytes.Buffer
        wrap := func() {
                err := executor.Stream(remotecommand.StreamOptions{
                        Stdout: &buf,
                        Tty: true,
                })
                done <- err
        }

        go wrap()

        fmt.Println("wait for out")

        select {
        case err := <-done:
                if err != nil {
                        fmt.Println("Command exit with error", err)
                }
                fmt.Println(string(buf.Bytes()))
        }
}
```

##	0x03	问题
笔者项目中遇到了在某些 kubernetes 集群版本下，当网关退出时，容器内的连接（网关到 kubernetes APIserver）、容器内 `bash` 残留的问题，如下描述：

1、`NewSPDYExecutor` 创建的 stream 泄漏问题：

关联相关 issue：
-	[How to cancel a SPDYExecutor stream? #554](https://github.com/kubernetes/client-go/issues/554)
-	[How to cancel a RESTClient exec? Can add context to the request？ #884](https://github.com/kubernetes/client-go/issues/884)

本项目中涉及到如下库：

```GO
go 1.17

require (
		// ......
        k8s.io/api v0.24.3
        k8s.io/apimachinery v0.24.3
        k8s.io/client-go v0.24.3
        k8s.io/kubectl v0.22.0
)
```

问题原因是通过 `exec.Stream` 构建的阻塞方法，开发者无法主动关闭这个 goroutine（网上也有相关 wrapped 改造，但是个人感觉不优雅），解决方案是使用新版本的 `client-go` 库，它提供了 `StreamWithContext`[方法](https://pkg.go.dev/k8s.io/client-go/tools/remotecommand)

```go
func main(){
	// ...
	exec, err := remotecommand.NewSPDYExecutor(config, "POST", req.URL())
	if err != nil {
		return err
	}
	err = exec.Stream(remotecommand.StreamOptions{
		Stdin:             stdin,
		Stdout:            stdout,
		Stderr:            stderr,
		Tty:               true,
		TerminalSizeQueue: terminalSizeQueue,
	})

	// ...
}
```

2、`bash` 残留

3、不当使用 `TerminalSizeQueue` 导致的内存泄漏问题

这里以 [v0.30.2](https://github.com/kubernetes/client-go/blob/v0.30.2/tools/remotecommand/v3.go) 版本分析下窗口动态更新的实现（`v3`），先看下用户实现的 `Next` 方法是在何处调用的，如下：

```go
// streamProtocolV3 implements version 3 of the streaming protocol for attach
// and exec. This version adds support for resizing the container's terminal.
type streamProtocolV3 struct {
	*streamProtocolV2

	resizeStream io.Writer		//resizeStream 是一个 io.Writer
}


func (p *streamProtocolV3) handleResizes() {
	if p.resizeStream == nil || p.TerminalSizeQueue == nil {
		// 这两个都需要定义才能启用
		return
	}
	go func() {
		defer runtime.HandleCrash()

		encoder := json.NewEncoder(p.resizeStream)
		for {
			// 在循环中调用 p.TerminalSizeQueue.Next() 拿到一个 size（当前的窗口尺寸）
			size := p.TerminalSizeQueue.Next()
			if size == nil {
				// 注意：协程退出的条件，需要外界主动触发
				return
			}
			if err := encoder.Encode(&size); err != nil {
				runtime.HandleError(err)
			}
		}
	}()
}

// stream：streamProtocolV3创建execstream入口
func (p *streamProtocolV3) stream(conn streamCreator) error {
        if err := p.createStreams(conn); err != nil {
                return err
        }

        // now that all the streams have been created, proceed with reading & copying

        errorChan := watchErrorStream(p.errorStream, &errorDecoderV3{})

		//默认在创建stream时，会异步启动handleResizes方法
        p.handleResizes()

        p.copyStdin()

        var wg sync.WaitGroup
        p.copyStdout(&wg)
        p.copyStderr(&wg)

        // we're waiting for stdout/stderr to finish copying
        wg.Wait()

        // waits for errorStream to finish reading with an error or nil
        return <-errorChan
}
```

这段代码可知，在 `handleResizes` 中异步方式启动并监听由 `Next` 方法传递的窗口信息，并将新的 size 信息编码为 JSON 格式发送给 `p.resizeStream`，特别注意这里 goroutine 退出的条件是 `size == nil`，所以外部在调用 `exec.StreamWithContext` 时一定要注意在合适的时机关闭该 goroutine，不然会造成内存泄漏

##  0x04	参考
-	[remotecommand 包](https://github.com/kubernetes/client-go/blob/master/tools/remotecommand)
-	[kubectl 的 resize 窗口实现](https://github.com/kubernetes/kubectl/blob/master/pkg/util/term/resize.go)
-	[kubernetes client-go web 命令行终端内存泄露问题解决](https://zhuanlan.zhihu.com/p/365431632)
-	[How to cancel a RESTClient exec? Can add context to the request？](https://github.com/kubernetes/client-go/issues/884)
-	[一个 Kubernetes Web 终端连接工具](https://jiankunking.com/kubernetes-client-go-how-to-make-a-web-terminal.html)
-	[Kubectl exec 的工作原理解读](https://icloudnative.io/posts/how-it-works-kubectl-exec/)
-	[利用 kubernetes exec 接口实现任意容器的 web-terminal](https://bbs.huaweicloud.com/blogs/281515)
-	[一个 Kubernetes Web 终端连接工具](https://jiankunking.com/kubernetes-client-go-how-to-make-a-web-terminal.html)
-	[remotecommand](https://github.com/kubernetes/client-go/blob/v0.30.2/tools/remotecommand/v3.go)
-	[kubectl进入node shell](https://www.huweihuang.com/article/kubernetes-notes/operation/kubectl/kubectl-node-shell/)