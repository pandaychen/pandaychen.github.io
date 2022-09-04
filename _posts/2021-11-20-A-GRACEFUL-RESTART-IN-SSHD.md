---
layout:     post
title:      Golang 系统编程：如何实现对后台服务优雅的热重启？
subtitle:   网关的热重启机制：原理&&实现&&分析（fvbock/endless）
date:       2021-11-20
author:     pandaychen
header-img:
catalog: true
category:   false
tags:
    - 热重启
    - Golang
---

##  0x00    前言

####    基本概念
热重启（Graceful Restart），是一项保证服务可用性的手段。它允许服务重启期间，不中断已经建立的连接，老服务进程不再接受新连接请求，新连接请求将在新服务进程中受理。对于原服务进程中已经建立的连接，也可以将其设为读关闭，等待平滑处理完连接上的请求及连接空闲后再行退出。通过这种方式，可以保证已建立的连接不中断，连接上的事务（请求、处理、响应）可以正常完成，新的服务进程也可以正常接受连接、处理连接上的请求

####    场景&&实现难点
热重启一般有如下场景：
-   分布式部署的服务可以通过网关/负载均衡器等控制流量来实现分批重启，从而达到服务热重启的目的
-   多进程Master-Worker模型的服务，如Nginx/SSHD等，可通过监听信号、Fork的方式重启
-   单进程的服务，可以通过拉起子进程后父进程退出的方式实现热重启

项目中笔者遇到实现的SSH网关，升级时面临到热重启的问题，即重启后，原先网关上的SSH长连接不可以断开（影响体验），并且网关要成功升级这个要求，如何实现呢？参考基于Linux-C实现的openssh-sshd源码，其网络的核心部分是由accept+fork经典模型实现的，主进程的重启不影响子进程（原有的ssh连接不会断），如何在go的服务中实现此特性呢？这就需要热重启技术，优雅热重启表现为两点：

1.  进程在不关闭其所监听的端口的情况下重启
2.  重启过程中保证所有请求能被正确的处理

这里的技术难点是：
-   如何实现热重启且先前的父进程等待所有的连接退出后，再退出
-   如何实现存量进程迁移？

##  0x01    涉及到系统编程知识
不过多描述，推荐[Linux系统编程](https://book.douban.com/subject/3907181/)
-   Fork机制
-   Reuseport机制：支持多个子进程可以监听在同一端口，进行`accept()`，笔者之前实现的简单TCP[框架](https://github.com/pandaychen/tcpframe)就基于此
-   Signal机制：通常用`SIGUSR1`、`SIGUSR2`等来实现自定义热重启的机制
-   ForkExec机制：如何启动一个进程（go和C不一样）：go标准库提供的函数是`syscall.ForkExec`而不是`syscall.Fork`
-   如何传递`fd`：已经存在于主进程的连接如何移动到新启动的子进程，如`listenFD`
-   子进程如何重建`listenFD`
-   父进程如何优雅退出？退出前保证当前父进程持有的长连接不能断开（等到这些连接退出后，父进程才可以退出）


####    项目中的方式
1.  用新的bin文件去替换老的bin文件
2.  发送信号告知server进程进行平滑升级
3.  server进程收到信号后，通过调用 `fork`/`exec` 启动新的版本的进程
4.  子进程调用接口获取从父进程继承的 socket 文件描述符重新监听 socket
5.  老的进程不再接受请求，待正在处理中的请求处理完后，进程自动退出
6.  子进程托管给init进程


####    Fork in go
使用`exec`包的 `Fork` 调用，并且使用 `cmd.ExtraFiles` 继承父进程已打开的文件，如下代码。通过调用 `exec` 包的 `Command` 命令传入 `path`（将要执行的命令路径）、`args` （命令的参数）即可返回 `exec.Command` 实例，通过 `ExtraFiles` 字段指定额外被新进程继承的已打开文件，最后调用 `cmd.Start` 方法创建子进程。

注意`cmd.ExtraFiles`的类型是`[]*os.File{}`

```Golang
func runPid(){
    //netListener 是父进程的listener
    file := netListener.File() // this returns a Dup()
    path := "/path/to/executable"
    args := []string{"-graceful"}
    // 产生 Cmd 实例
    cmd := exec.Command(path, args...)
    // 标准输出
    cmd.Stdout = os.Stdout
    // 标准错误输出
    cmd.Stderr = os.Stderr
    cmd.ExtraFiles = []*os.File{file}       //继承netListener
    // 启动命令
    err := cmd.Start()
    if err != nil {
        log.Fatalf("Failed to launch, error: %v", err)
    }
}
```

上面`netListener.File`会通过系统调用 `dup` 复制一份 file descriptor，此新的文件描述符和参数 oldfd 指向同一个文件，共享所有的索性、读写指针、各项权限或标志位等。但是不共享关闭标志位，也就是说 oldfd 已经关闭了，也不影响写入新的数据到 newfd 中，如下图（子进程复制父进程的文件描述符表）：
```golang
func Dup(oldfd int) (fd int, err error) {
    r0, _, e1 := Syscall(SYS_DUP, uintptr(oldfd), 0, 0)
    fd = int(r0)
    if e1 != 0 {
        err = errnoErr(e1)
    }
    return
}
```

![fork-dup](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/system/restart/fork-dup-1.png)

####	REUSEPORT == 平滑重启？
reuseport是为了充分利用多核，提升连接建立的效率。先说结论：新旧进程使用reuseport机制监听同一地址是无法做到无损的平滑重启

reuseport特性可以让多个进程/线程监听同一个ip:port，每监听一次就会新建一个新的listen socket，每个socket都有自己的半连接和全连接队列。内核将不同用户的握手请求随机分配到不同的socket的半连接队列，完成了完整的握手流程后再进入半连接所在套接字对应的全连接队列中，等待accept

启用reuseport下触发热重启时，在子进程里创建一个新的socket。由于父进程的套接字有自己的半连接和全连接队列，新子进程的套接字也有自己的半连接和全连接队列。父进程停止accept，只处理已经accept的历史连接再退出服务，那么父进程的全连接队列中未被accept的连接就丢失了，也就实现不了无损平滑重启了。如果父进程不停止accept，那么kernel还是会源源不断把部分请求分配给父进程的套接字，这样父进程不退出，也就不能实现服务的更新

####    如何平滑重启？
重启时处理已建立连接上请求？有N种选择：
-   shutdown read，不再接受新的请求，对端继续写数据的时候会感知到失败
-   继续处理连接上已经正常接收的请求，处理完成后Close连接（也可能是客户端主动close）
-   不主动进行读端关闭，而是等连接空闲一段时间后再close
-   高级方式是将connfd、connfd上已经读取写入的buffer数据也一并传递给子进程处理
-   针对消息类服务，确认下自己服务的消息消费、确认机制是否合理（不再收新消息、处理完已收到的消息后，再退出）

##  0x02    开源项目比较（go）
####    方案一
经典方式，通过监听信号，收到信号后fork子进程，新请求都由子进程处理，父进程等正在处理的服务处理完成后退出，典型的库有[endless](https://github.com/fvbock/endless)、[grace](https://github.com/facebookarchive/grace)
1.  监听信号，父进程收到信号时 fork 子进程（使用相同的启动命令），将服务监听的 socket 文件（`listenFD`）描述符传递给子进程
2.  子进程监听父进程的 socket，这个时候父进程和子进程都可以接收请求
3.  子进程启动成功之后，父进程停止接收新的连接，等待旧连接处理完成（或超时）
4.  父进程退出，升级完成


####    方案二
使用主进程来检查和安装升级，子进程处理请求，重启的时候主进程fork新的子进程，新请求由新子进程处理，旧子进程等待正在处理的服务处理完后退出，典型的库有[overseer](https://github.com/jpillora/overseer)

1.  监听信号，收到信号时 fork 新的子进程（使用相同的启动命令），将服务监听的 socket 文件描述符传递给新的子进程
2.  子进程监听父进程的 socket，这个时候旧子进程和新子进程都可以接收请求
3.  新子进程启动成功之后，旧子进程停止接收新的连接，等待旧连接处理完成（或超时）
4.  旧子进程退出，升级完成
5.	overseer 增加了一个主进程管理平滑重启。子进程处理连接，能够保持主进程 `pid` 不变

![overseer](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/system/restart/overseer.png)


##  0x03    endless分析
[endless用法](https://www.luozhiyun.com/archives/584)参考此文，测试了下，此库满足如下场景：当热重启（通过`kill -SIGHUP pid`触发）后父进程并不是立即退出，而是在父进程所有管理的连接全部完成后才退出，本小节分析下其热重启的实现：

```golang
//控制退出
func handler(w http.ResponseWriter, r *http.Request) {
    duration, err := time.ParseDuration(r.FormValue("duration"))
    if err != nil {
        http.Error(w, err.Error(), 400)
        return
    }
    time.Sleep(duration)
    w.Write([]byte("Hello World"))
}
```

####    endless的原理
1.  监听 `SIGHUP` 信号
2.  收到信号时 fork 子进程（使用相同的启动命令），将服务监听的 socket 文件描述符传递给子进程；
子进程监听父进程的 socket，这个时候父进程和子进程都可以接收请求
3.  子进程启动成功之后发送 SIGTERM 信号给父进程，父进程停止接收新的连接，等待旧连接处理完成（或超时）
4.  父进程退出，升级完成

![principle](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/system/restart/endless-principle.png)

####    核心结构
```GOLANG
type endlessServer struct {
	http.Server     //继承 http.Server 结构
	EndlessListener  net.Listener   // 监听客户端请求的 Listener
	SignalHooks      map[int]map[os.Signal][]func()
	tlsInnerListener *endlessListener
	wg               sync.WaitGroup  // 用于记录还有多少客户端请求没有完成
	sigChan          chan os.Signal  // 用于接收信号的管道
	isChild          bool        // 用于重启时标志本进程是否是为一个新进程
	state            uint8      // 当前进程的状态
	lock             *sync.RWMutex
	BeforeBegin      func(add string)
}   
```

注意上面结构体中的`wg`的用途，解决了上面说到的必须等所有连接都退出，父进程才退出的作用。

[初始化代码](https://github.com/fvbock/endless/blob/master/endless.go#L89)如下，这里需要注意，初始化方法中通过 `ENDLESS_CONTINUE` 环境变量来判断是否是个子进程，此变量会在 fork 子进程的时候写入。此外，由于 endless 是支持多 server 的，需要用 `ENDLESS_SOCKET_ORDER`变量来判断一下 server 的顺序
```golang
func NewServer(addr string, handler http.Handler) (srv *endlessServer) {
	runningServerReg.Lock()
	defer runningServerReg.Unlock()

     // 根据环境变量判断是不是子进程
	socketOrder = os.Getenv("ENDLESS_SOCKET_ORDER")
	isChild = os.Getenv("ENDLESS_CONTINUE") != ""   //fork方法中，会设置子进程的ENDLESS_CONTINUE==1

    // 由于支持多 server，所以这里需要设置一下 server 的顺序
	if len(socketOrder) > 0 {
		for i, addr := range strings.Split(socketOrder, ",") {
			socketPtrOffsetMap[addr] = uint(i)
		}
	} else {
		socketPtrOffsetMap[addr] = uint(len(runningServersOrder))
	}

    //构建
	srv = &endlessServer{
		wg:      sync.WaitGroup{},
		sigChan: make(chan os.Signal),
		isChild: isChild,
        //
		SignalHooks: map[int]map[os.Signal][]func(){
			PRE_SIGNAL: map[os.Signal][]func(){
				syscall.SIGHUP:  []func(){},
				syscall.SIGUSR1: []func(){},
				syscall.SIGUSR2: []func(){},
				syscall.SIGINT:  []func(){},
				syscall.SIGTERM: []func(){},
				syscall.SIGTSTP: []func(){},
			},
			POST_SIGNAL: map[os.Signal][]func(){
				syscall.SIGHUP:  []func(){},
				syscall.SIGUSR1: []func(){},
				syscall.SIGUSR2: []func(){},
				syscall.SIGINT:  []func(){},
				syscall.SIGTERM: []func(){},
				syscall.SIGTSTP: []func(){},
			},
		},
		state: STATE_INIT,
		lock:  &sync.RWMutex{},
	}

	srv.Server.Addr = addr
	srv.Server.ReadTimeout = DefaultReadTimeOut
	srv.Server.WriteTimeout = DefaultWriteTimeOut
	srv.Server.MaxHeaderBytes = DefaultMaxHeaderBytes
	srv.Server.Handler = handler

	srv.BeforeBegin = func(addr string) {
		log.Println(syscall.Getpid(), addr)
	}

	runningServersOrder = append(runningServersOrder, addr)
	runningServers[addr] = srv

	return
}
```

####    Listen and Serve
`ListenAndServe`提供的对外的支持热重启的方法，好奇`if srv.isChild{...}`的用法？
```GOLANG
/*
ListenAndServe listens on the TCP network address srv.Addr and then calls Serve
to handle requests on incoming connections. If srv.Addr is blank, ":http" is
used.
*/
func (srv *endlessServer) ListenAndServe() (err error) {
	addr := srv.Addr
	if addr == "" {
		addr = ":http"
	}
    // 异步处理信号量
	go srv.handleSignals()
    // 获取端口监听
	l, err := srv.getListener(addr)
	if err != nil {
		log.Println(err)
		return
	}
    // 将监听转为 endlessListener
	srv.EndlessListener = newEndlessListener(l, srv)

    // 如果是子进程，那么发送 SIGTERM 信号给父进程
	if srv.isChild {
		syscall.Kill(syscall.Getppid(), syscall.SIGTERM)
	}

	srv.BeforeBegin(srv.Addr)
    // 响应Listener监听，执行对应请求逻辑
	return srv.Serve()
}
```

上面的`handleSignals`是注册待处理的signals，[实现如下](https://github.com/fvbock/endless/blob/master/endless.go#L313)，重点分析`syscall.SIGHUP`信号的处理过程：
```golang
/*
handleSignals listens for os Signals and calls any hooked in function that the
user had registered with the signal.
*/
func (srv *endlessServer) handleSignals() {
	var sig os.Signal

	signal.Notify(
		srv.sigChan,
		hookableSignals...,
	)

    //获取当前pid
	pid := syscall.Getpid()
	for {
		sig = <-srv.sigChan
        //前置， 在处理信号之前触发hook
		srv.signalHooks(PRE_SIGNAL, sig)
		switch sig {
		case syscall.SIGHUP:
			log.Println(pid, "Received SIGHUP. forking.")
            //核心逻辑全部在此，接收到平滑重启信号
			err := srv.fork()
			if err != nil {
				log.Println("Fork err:", err)
			}
		case syscall.SIGUSR1:
			log.Println(pid, "Received SIGUSR1.")
		case syscall.SIGUSR2:
			log.Println(pid, "Received SIGUSR2.")
			srv.hammerTime(0 * time.Second)
		case syscall.SIGINT:
			log.Println(pid, "Received SIGINT.")
			srv.shutdown()
		case syscall.SIGTERM:
			log.Println(pid, "Received SIGTERM.")
			srv.shutdown()
		case syscall.SIGTSTP:
			log.Println(pid, "Received SIGTSTP.")
		default:
			log.Printf("Received %v: nothing i care about...\n", sig)
		}
        //后置， 在处理信号之后触发hook
		srv.signalHooks(POST_SIGNAL, sig)
	}
}
```

![signals](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/system/restart/endless-signals.png)

####    接收SIGHUP信号后fork流程（核心！）
在上面进入到`case syscall.SIGHUP` 调用 fork 方法，首先会根据 server 来获取不同的 listen fd 然后封装到 `files` 列表中，然后在调用 `cmd` 的时候将文件描述符传入到 `ExtraFiles` 参数中，这样子进程就可以无缝托管到父进程监听的端口：
```GO
func (srv *endlessServer) fork() (err error) {
    runningServerReg.Lock()
    defer runningServerReg.Unlock()

    // 校验是否已经fork过
    if runningServersForked {
        return errors.New("Another process already forked. Ignoring this one.")
    } 
    runningServersForked = true

    var files = make([]*os.File, len(runningServers))
    var orderArgs = make([]string, len(runningServers))
    // 因为有多 server 的情况，所以获取所有 listen fd
    for _, srvPtr := range runningServers { 
        switch srvPtr.EndlessListener.(type) {
        case *endlessListener: 
            files[socketPtrOffsetMap[srvPtr.Server.Addr]] = srvPtr.EndlessListener.(*endlessListener).File()
        default: 
            files[socketPtrOffsetMap[srvPtr.Server.Addr]] = srvPtr.tlsInnerListener.File()
        }
        orderArgs[socketPtrOffsetMap[srvPtr.Server.Addr]] = srvPtr.Server.Addr
    }
    // 环境变量
    env := append(
        os.Environ(),
    // 启动endless 的时候，会根据这个参数来判断是否是子进程
        "ENDLESS_CONTINUE=1",
    )

    //将所有的server信息，放入ENV
    if len(runningServers) > 1 {
        env = append(env, fmt.Sprintf(`ENDLESS_SOCKET_ORDER=%s`, strings.Join(orderArgs, ",")))
    }

    // 程序运行路径
    path := os.Args[0]
    var args []string
    // 参数
    if len(os.Args) > 1 {
        args = os.Args[1:]
    }

    cmd := exec.Command(path, args...)
    // 标准输出
    cmd.Stdout = os.Stdout
    // 错误
    cmd.Stderr = os.Stderr
    cmd.ExtraFiles = files      //继承父进程的fd
    cmd.Env = env  
    err = cmd.Start()
    if err != nil {
        log.Fatalf("Restart: Failed to launch, error: %v", err)
    } 
    return
}
```

####    接收SIGTERM信号后shutdown流程
现网使用较少，这种方法过于暴力；`shutdown`方法会先将连接关闭，因为这个时候子进程已经启动了，所以不再处理请求，需要把端口的监听关了。这里还会异步调用 `srv.hammerTime` 方法等待`60`秒把父进程的请求处理完毕才关闭父进程。
```golang
func (srv *endlessServer) shutdown() {
    if srv.getState() != STATE_RUNNING {
        return
    }

    srv.setState(STATE_SHUTTING_DOWN)
    // 默认 DefaultHammerTime 为 60秒
    if DefaultHammerTime >= 0 {
        go srv.hammerTime(DefaultHammerTime)
    }
    // 关闭存活的连接
    srv.SetKeepAlivesEnabled(false)
    err := srv.EndlessListener.Close()
    if err != nil {
        log.Println(syscall.Getpid(), "Listener.Close() error:", err)
    } else {
        log.Println(syscall.Getpid(), srv.EndlessListener.Addr(), "Listener closed.")
    }
}

func (srv *endlessServer) hammerTime(d time.Duration) {
	defer func() {
		// we are calling srv.wg.Done() until it panics which means we called
		// Done() when the counter was already at 0 and we're done.
		// (and thus Serve() will return and the parent will exit)
		if r := recover(); r != nil {
			log.Println("WaitGroup at 0", r)
		}
	}()
	if srv.getState() != STATE_SHUTTING_DOWN {
		return
	}
	time.Sleep(d)
	log.Println("[STOP - Hammer Time] Forcefully shutting down parent")
	for {
		if srv.getState() == STATE_TERMINATE {
			break
		}
		srv.wg.Done()
		runtime.Gosched()
	}
}
```

####    getListener 获取端口监听（核心！）
针对父子进程的处理是不一样的，分析下子进程重建listener的流程（下面代码有一个魔数`3`）：

`os.NewFile` 的参数为何从`3`开始？因为子进程在继承父进程的 fd 的时候预留`0-STDIN`，`1-STDOUT`，`2-STDERR`，所以父进程给的第一个fd在子进程里顺序排就是从`3`开始，又因为 fork 的时候`cmd.ExtraFiles` 参数传入的是一个 `files`，如果有多个 server 那么会依次从`3`开始递增。

```GOLANG
func (srv *endlessServer) getListener(laddr string) (l net.Listener, err error) {
    // 如果是子进程
    if srv.isChild {
        var ptrOffset uint = 0
        runningServerReg.RLock()
        defer runningServerReg.RUnlock()
        // 这里还是处理多个 server 的情况
        if len(socketPtrOffsetMap) > 0 {
            // 根据server 的顺序来获取 listen fd 的序号
            ptrOffset = socketPtrOffsetMap[laddr] 
        }
        // fd 0，1，2是预留给 标准输入、输出和错误的，所以从3开始
        f := os.NewFile(uintptr(3+ptrOffset), "")
        l, err = net.FileListener(f)
        if err != nil {
            err = fmt.Errorf("net.FileListener error: %v", err)
            return
        }
    } else {
        // 父进程 直接返回 listener
        l, err = net.Listen("tcp", laddr)
        if err != nil {
            err = fmt.Errorf("net.Listen error: %v", err)
            return
        }
    }
    return
}
```

如下图展示了fd的映射，`3` 是根据传入 `ExtraFiles` 的数组列表依次递增：
![dup-2](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/system/restart/fork-dup-2.png)

####    父进程何时退出？
本质上还是通过`sync.WaitGroup`机制来保证该功能：

-   Accept[方法](https://github.com/fvbock/endless/blob/master/endless.go#L489)：`wg.Add(1)`，接收新连接时计数加`1`
-   Close[方法](https://github.com/fvbock/endless/blob/master/endless.go#L537)：`w.server.wg.Done()`，连接关闭时，计数减`1`
-   Serve[方法](https://github.com/fvbock/endless/blob/master/endless.go#L192)：`srv.wg.Wait()`，所有的启动监听都会调用该方法，阻塞在`srv.Server.Serve(srv.EndlessListener)`，当热重启父进程退出前，阻塞在`srv.wg.Wait()`上等待所有子goroutine退出

```golang
func (el *endlessListener) Accept() (c net.Conn, err error) {
	tc, err := el.Listener.(*net.TCPListener).AcceptTCP()
	if err != nil {
		return
	}

	tc.SetKeepAlive(true)                  // see http.tcpKeepAliveListener
	tc.SetKeepAlivePeriod(3 * time.Minute) // see http.tcpKeepAliveListener

	c = endlessConn{
		Conn:   tc,
		server: el.server,
	}

    //add 1
	el.server.wg.Add(1)
	return
}

func (w endlessConn) Close() error {
	err := w.Conn.Close()
	if err == nil {
        //clone连接退出时，Done
		w.server.wg.Done()
	}
	return err
}
```

在启动的方法中，等待子进程结束才退出：
```GOLANG

/*
Serve accepts incoming HTTP connections on the listener l, creating a new
service goroutine for each. The service goroutines read requests and then call
handler to reply to them. Handler is typically nil, in which case the
DefaultServeMux is used.
In addition to the stl Serve behaviour each connection is added to a
sync.Waitgroup so that all outstanding connections can be served before shutting
down the server.
*/
func (srv *endlessServer) Serve() (err error) {
	defer log.Println(syscall.Getpid(), "Serve() returning...")
	srv.setState(STATE_RUNNING)
	err = srv.Server.Serve(srv.EndlessListener)
	log.Println(syscall.Getpid(), "Waiting for connections to finish...")

    //等待所有的连接完成
	srv.wg.Wait()
	srv.setState(STATE_TERMINATE)

    //父进程真正退出了
	return
}
```

`STATE_TERMINATE`的作用是什么呢？

##  0x04    teleport的实现
[Restarting a Go Program Without Downtime](https://goteleport.com/blog/golang-ssh-bastion-graceful-restarts/)一文也给了详细的实现原理，代码示例[在此](https://github.com/pandaychen/golang_in_action/blob/master/system/graceful_restart/teleport_example.go)

示例代码给了`SIGUSR2`以及`SIGQUIT`的分别用法：

1、`SIGUSR2`<br>
使用reuseport特性，当使用`kill -SIGUSR2`重启时，父子进程都可以收到请求（监听在同一端口），此时需要手动把父进程关闭
![SIGUSR2](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/system/ssh-graceful-restart1.png)

2、`SIGQUIT`<br>
使用`SIGQUIT`时，父进程等待指定时间后，退出（通过`context.WithTimeout`）
```golang
// Fork a child process.
p, err := forkChild(addr, ln)
if err != nil {
	fmt.Printf("Unable to fork child: %v.\n", err)
	continue
}
fmt.Printf("Forked child %v.\n", p.Pid)

// Create a context that will expire in 5 seconds and use this as a timeout to Shutdown.
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()

// Return any errors during shutdown.
return server.Shutdown(ctx)
```

![SIGQUIT](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/system/ssh-graceful-restart2.png)

不过，上面两种方式都不够优雅。

##  0x05    其他库的机制介绍


##  0x06  总结
endless的实现方式，可以保证已建立的连接不中断，新的服务进程也可以正常接受连接请求，项目中值得借鉴。


##  0x07  参考
-   [Go如何实现热重启](https://zhuanlan.zhihu.com/p/230888784)
-   [endless 如何实现不停机重启 Go 程序？](https://www.luozhiyun.com/archives/584)
-   [Restarting a Go Program Without Downtime](https://goteleport.com/blog/golang-ssh-bastion-graceful-restarts/)
-   [平滑重启](https://learnku.com/articles/23505/graceful-restart)
-   [Zero downtime restarts for go servers (Drop in replacement for http.ListenAndServe)](https://github.com/fvbock/endless)
-   [Graceful process restarts in Go](https://github.com/cloudflare/tableflip)
-   [Graceful upgrades in Go](https://blog.cloudflare.com/graceful-upgrades-in-go/)
-   [go优雅升级/重启工具调研](https://lrita.github.io/2019/06/06/golang-graceful-upgrade/#%E5%8F%82%E8%80%83)