---
layout:     post
title:      一种基于 TTY-based 的 kubernetes console 实现
subtitle:   如何使用 ssh 协议打通 kubernetes EXEC 登录？
date:       2022-06-01
author:     pandaychen
catalog:    true
tags:
    - Kubernetes
---

##  0x00    开篇
网上基于 [websocket](https://github.com/gorilla/websocket) 打通 kubernetes pod 的实现非常多，但是受限于 webconsole 的不便利性，这边文章来分析下如何使用 SSH 方式打通 kubernetes 的登录。


##  0x01    实现思路

1、remotecommand暴露的`Executor`方法<br>

我们实现的核心是打通IO（即`stdin`与`stdout`）之间的桥接，如下图：

![flow](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/ssh/ssh2kubernetesflow.png)

kubernetes的[client-go](https://github.com/kubernetes/client-go/blob/master/tools/remotecommand/remotecommand.go)包提供了集群中的容器建立长连接的方法，并设置容器的 `stdin`，`stdout` 等。此package提供了基于 SPDY 协议的 `Executor`，其本质是一个 `interface`，用于和 pod 终端的流的传输及通信。`Executor` 接口定义了 `Stream` [方法](https://github.com/kubernetes/client-go/blob/master/tools/remotecommand/remotecommand.go#L108)，该方法会建立一个流传输的连接，直到服务端和调用端一端关闭连接，才会停止传输。（原package中给出的实现是基于HTTP-SPDY的，不适用于我们的场景，所有我们需要重新实现`Stream`方法）

此外，考虑到`StreamOptions`是`Stream`方法的参数，在实现`Stream`方法时需要填充该结构。

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

关于`SteamOption`中各个成员的作用，可以参见[V1](https://github.com/kubernetes/client-go/blob/master/tools/remotecommand/v1.go)的实现；简言之，就是外部对接容器内部暴露的输入/输出。

####	TerminalSizeQueue结构
TerminalSizeQueue is capable of returning terminal resize events as they occur.
```golang
type TerminalSizeQueue interface {
	// Next returns the new terminal size after the terminal has been resized. It returns nil when
	// monitoring has been stopped.
	Next() *TerminalSize
}
```


2、定义核心结构体，构造`StreamOptions`所需的成员<br>
```golang
// TtyHandler
type PyStreamOption interface {
	io.Reader	//stdin
	io.Writer	//stdout && stderr
	//io.Writer	
	remotecommand.TerminalSizeQueue
}
```
定义如上结构`PyStreamOption`，然后需要实例化`PyStreamOption`；注意实例化之后，我们必须实现该 `interface`的`3`个方法：

-	`Read(p []byte) (int, error)`：从前置SSH会话（ssh.Channel）中读取数据，转发到用户侧
-	`Write(p []byte) (int, error)`：从前置SSH会话获取用户输入，写入pod
-	`Next() *TerminalSize`：即时调整终端的size

设置的目的是：
1.	对接terminal的输入输出
2.	实时监听窗口大小的改变并调整

这样，后续调用 `Stream` 方法时，只要将 `StreamOptions` 的 `Stdin`/`Stdout` 都设置为 `PyStreamOption`，`Executor` 会通过你定义的 `Write` 和 `Read` 方法来传输数据。

3、建立容器（连接）的方法<br>
注意下面的`CreateK8SStream`方法，最后会使用`remotecommand`的`exec.Stream`方法来实现打通容器登录的过程（`stdin`/`stdout`/`stderr`由参数传入），即上一步定义的`TtyHandler`结构的相关成员：
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
		Stdin:             stdin,		//来自option
		Stdout:            stdout,		//来自option
		Stderr:            stderr,		//来自option
		Tty:               true,		//来自option，设置TTY
		TerminalSizeQueue: terminalSizeQueue,	//来自option
	})
	//...
}
```

接下来的思路就是如何生成一个`Executor`，来打通SSH到SPDY（主要还是打通上图的STDIN/STDOUT）。


##  0x02	参考
-	[remotecommand包](https://github.com/kubernetes/client-go/blob/master/tools/remotecommand)