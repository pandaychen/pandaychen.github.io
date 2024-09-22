---
layout:     post
title:      主机入侵检测系统 Elkeid：设计与分析（一）
subtitle:   后台服务模块
date:       2024-08-19
author:     pandaychen
catalog:    true
tags:
    - HIDS
    - ELKEID
---


##  0x00    前言
Elkeid 后台主要模块如下：

-   AgentCenter 模块：负责与 Agent 进行通信，采集 Agent 数据并简单处理后汇总到消息队列集群，同时也负责对 Agent 进行管理包括 Agent 的升级，配置修改，任务下发等。同时 AgentCenter 也对外提供 HTTP 接口，Manager 模块通过这些 HTTP 接口实现对 AgentCenter 和 Agent 的管理和监控
-   ServiceDiscovery 模块：注册中心，后台中的各个服务模块都需要向 ServiceDiscovery 定时注册、同步服务信息，从而保证各个服务模块中的实例相互可见，便于直接通信。由于 ServiceDiscovery 模块维护了各个注册服务的状态信息，所以当服务使用方在请求服务发现时，ServiceDiscovery 模块会进行负载均衡。比如 Agent 请求获取 AgentCenter 模块实例列表，ServiceDiscovery 模块直接返回负载压力最小的 AgentCenter 实例
-   Manager 模块：负责对整个后台进行管理并提供相关的查询、管理接口。包括管理 AgentCenter 集群，监控 AgentCenter 状态，控制 AgentCenter 服务相关参数，并通过 AgentCenter 管理所有的 Agent，收集 Agent 运行状态，向 Agent 下发任务（走控制 control 信道），同时 manager 模块也负责管理实时和离线计算集群
-   Elkeid Console：Elkeid 前端部分，通过 Elkeid Console 可查看告警 / 资产数据
-   Elkeid HUB：Elkeid HIDS RuleEngine，对数据进行分析和检测（暂未开源）

本文基于 [v1.9.1.9](https://github.com/bytedance/Elkeid/releases/tag/v1.9.1.9) 版本分析，本文主要关注如下模块的实现

![backend](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/elkeid/arch/agent2agentcenter.png)


##  0x01    Agent
Elkeid Agent 的设计还是非常典型的，为笔者丰富了此类主机 Agent 开发视角。Elkeid Agent 提供端上组件的基本能力支撑，包括数据通信（RPC stream）、资源监控、组件版本控制、文件传输、机器基础信息采集等。Agent 本身作为一个插件底座以系统服务的方式运行，其他子插件的策略存放于服务器端的配置，Agent 接收到相应的控制指令及配置后对自身及插件进行开启、关闭、升级等操作

Agent 与 AgentServer 之间采用 bi-stream gRPC（TLS） 进行 [通信](https://github.com/pandaychen/elkeid_fork/blob/main/server/agent_center/grpctrans/proto/grpc.proto#L102)，一些细节如下：

1.  Agent -> AgentServer 方向为数据通道（上报），子 plugin 获取的数据会通过此链路上报
2.  AgentServer -> Agent 方向为指令通道（控制流），作为通知 Agent 执行相关的指令等，见 [此](https://github.com/pandaychen/elkeid_fork/blob/main/server/agent_center/grpctrans/proto/grpc.proto#L31C9-L31C19)，使用 protobuf 的不同 message 类型来标识不同指令类型
3.  Agent 本身支持客户端侧服务发现，也支持跨 Region 级别的通信配置，实现一个 Agent 包在多个网络隔离的环境下进行安装，基于底层一个 TCP 连接，在上层实现了 `Transfer` 与 `FileOp` 两种数据传输服务，支撑了插件本身的数据上报与 Host 中的文件交互
4.  Plugins（安全能力插件）与 Agent 为 **父/子** 进程关系，以两个 `os.Pipe()` 作为跨进程通信方式，Plugins 实现了 Go/Rust 的两个插件库，负责插件端信息的编码与发送（参考下图）。Plugins 发送数据后，会被编码为 Protobuf 二进制数据，Agent 接收到后无需二次解编码，在其外层拼接好 `Header` 特征数据，直接传输给 Server，一般情况下 Server 也无需解码，直接传输至后续数据流中，使用时进行解码，一定程度上降低了数据传输中多次编解码造成的额外性能开销，这里的逻辑参考 `WriteEncodedRecord` 的 [实现](https://github.com/pandaychen/elkeid_fork/blob/main/agent/plugin/plugin_linux.go#L158)，这里的实现思路非常值得借鉴
5.  Agent 采用 Go 开发，在 Linux 下，通过 `systemd` 作为守护方式，通过 `cgroup` 限制资源使用

![linux-os-pipe](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/elkeid/arch/os-pipe-agent-plugins.png)

##  0x02 Agent核心代码
Agent主要包含三个子模块

-   heartbeat：处理自身host层面/各个插件的核心运行指标上报
-   plugin：处理各个子插件的启动/停止（所有的插件），负责与各个子插件的上行（指令），下行（数据）通信
-   transport：负责与AgentServer通信，作为stream RPC的客户端，负责打通plugin模块与AgentServer的上下行传输通道

####    heartbeat

####    plugin：插件处理

1、插件的创建及加载

在`Startup`[方法](https://github.com/pandaychen/elkeid_fork/blob/main/agent/plugin/plugin.go#L172)中，通过接收来自AgentServer的RPC响应来执行插件的启停，插件的启停核心代码如下：

```GO
func Load(ctx context.Context, config proto.Config) (plg *Plugin, err error) {  
    //通过插件名称查找插件对象，并校验参数，校验失败则返回
    // m      = &sync.Map{} 
    loadedPlg, ok := m.Load(config.Name) 
    
    //...... 
    
    workingDirectory := path.Join(agent.WorkingDirectory, "plugin", config.Name)
    
    //构造插件的执行路径
    execPath := path.Join(workingDirectory, config.Name) 
    
    cmd := exec.Command(execPath)
    var rx_r, rx_w, tx_r, tx_w *os.File
    
    //创建插件进程->agent进程方向的管道，插件向rx_w写入数据，agent从rx_r中读取数据
    rx_r, rx_w, err = os.Pipe() 
    if err != nil {
        return
    }
    
    //创建agent进程->插件进程方向的管道，agent向tx_w写入数据，插件从tx_r中读取数据
    tx_r, tx_w, err = os.Pipe() 
    if err != nil {
        return
    }
    cmd.SysProcAttr = &syscall.SysProcAttr{Setpgid: true}
    
    //通过cmd.ExtraFiles将插件进程的标准输入stdin设置为tx_r，将插件进程的标准输出stdout设置为rx_w
    cmd.ExtraFiles = append(cmd.ExtraFiles, tx_r, rx_w) 
    cmd.Dir = workingDirectory
    var errFile *os.File
    errFile, err = os.OpenFile(execPath+".stderr", os.O_CREATE|os.O_RDWR|os.O_TRUNC, 0o0600)
    if err != nil {
        return
    }
    defer errFile.Close()
    cmd.Stderr = errFile //设置stderr为文件
    if config.Detail != "" {
        cmd.Env = append(cmd.Env, "DETAIL="+config.Detail)
    }
    logger.Info("plugin's process will start")
    err = cmd.Start()
    tx_r.Close()    //agent进程使用tx_w和rx_r，用不到tx_r和rx_w，所以将这两者关闭
    rx_w.Close()
    //......
}
```

在通过`cmd.Start()`以fork方式启动了插件子进程之后，接下来就是构造子插件，并且使用`os.Pipe`提供的读写机制进行通信，核心代码如下，包含了3个独立的子goroutine：

1.  异步等待插件plugin退出
2.  plugin->agent：采集数据上报
3.  agent->plugin：任务发送到plugin

```GO
func Load(ctx context.Context, config proto.Config) (plg *Plugin, err error) {

    // 此处省略前面的创建过程
    plg = &Plugin{
        Config:        config,
        mu:            &sync.Mutex{},
        cmd:           cmd,
        rx:            rx_r,        //plugin的读端
        updateTime:    time.Now(),
        reader:        bufio.NewReaderSize(rx_r, 1024*128), //将管道的读端初始化为带缓冲区的io.Reader对象，提高读效率
        tx:            tx_w,        //plugin的写端
        done:          make(chan struct{}),     //用于完成通知调用方的channel，参考<异步等待插件退出>实现
        taskCh:        make(chan proto.Task),   //传递任务的channel，异步
        wg:            &sync.WaitGroup{},
        SugaredLogger: logger,
    }
    plg.wg.Add(3)
    
    //等待插件进程退出的协程
    //退出时关闭rx_r和tx_w管道，同时将完成通知投递到plg.done channel中
    go func() {
        defer plg.wg.Done()
        defer plg.Info("gorountine of waiting plugin's process will exit")
        err = cmd.Wait() //等待插件进程退出（以阻塞形式等待插件进程退出）
        rx_r.Close()    //退出时关闭rx_r和tx_w管道
        tx_w.Close()
        if err != nil {
            plg.Errorf("plugin has exited with error:%v,code:%d", err, cmd.ProcessState.ExitCode())
        } else {
            plg.Infof("plugin has exited with code %d", cmd.ProcessState.ExitCode())
        }
        close(plg.done) //通知完成（发送插件完成通知）
    }()
    
    //异步接收插件数据，并发送到上层
    go func() { 
        defer plg.wg.Done()
        defer plg.Info("gorountine of receiving plugin's data will exit")
        for {
            rec, err := plg.ReceiveData()  //解析Record结构
            if err != nil {
                if errors.Is(err, bufio.ErrBufferFull) {
                    plg.Warn("when receiving data, buffer is full, skip this record")
                    continue
                } else if !(errors.Is(err, io.EOF) || errors.Is(err, io.ErrClosedPipe) || errors.Is(err, os.ErrClosed)) {
                    plg.Error("when receiving data, an error occurred: ", err)
                } else {
                    break
                }
            }
            // 这个方法实际上是将rec数据临时存储在上层的缓冲数组Buf中，为了提高上层的发送效率
            //  Buf               = [8192]interface{}{} 
            core.Transmission(rec, true) 
        }
    }()

    go func() { //异步向插件发送任务
        defer plg.wg.Done()
        defer plg.Info("gorountine of sending task to plugin will exit")
        for {
            select {
            case <-plg.done:
                return
            case task := <-plg.taskCh: //任务通道中有任务存在，取出任务
                s := task.Size()
                var dst = make([]byte, 4+s)
                _, err = task.MarshalToSizedBuffer(dst[4:])//将task任务序列化到dst缓冲区(从下标4开始)
                if err != nil {
                    plg.Errorf("when marshaling a task, an error occurred: %v, ignored this task: %+v", err, task)
                    continue
                }
                binary.LittleEndian.PutUint32(dst[:4], uint32(s))//task任务大小写入到dst的前4个字节中
                var n int
                n, err = plg.tx.Write(dst) //任务写入管道
                if err != nil {
                    if !errors.Is(err, os.ErrClosed) {
                        plg.Error("when sending task, an error occurred: ", err)
                    }
                    return
                }
                atomic.AddUint64(&plg.rxCnt, 1)
                atomic.AddUint64(&plg.rxBytes, uint64(n))
            }
        }
    }()
    m.Store(config.Name, plg) //将插件信息保存到sync.Map中
    return
}
```

附带一下`core.Transmission(rec, true)`的实现（并发安全）：

```GO
func Transmission(rec interface{}, tolerate bool) (err error) {
    if hook != nil {
        rec = hook(rec)
    }
    
    //加Mutex锁，线程安全的写Buf
    Mu.Lock()
    defer Mu.Unlock()
    if Offset < len(Buf) {
        Buf[Offset] = rec
        Offset++ //偏移量递增
        return
    }
    if tolerate {
        err = ErrBufferOverflow
    } else {
        Buf[0] = rec
    }
    return
}
```

3、插件的模版实现（Golang）

elkeid对所有插件与Agent的通信做了封装，代码比较直观，[参考](https://github.com/pandaychen/elkeid_fork/tree/main/plugins/lib/go)，核心就是`Client.SendRecord`和`Client.ReceiveTask`方法：


```GO
func New() (c *Client) {
    c = &Client{
        rx: os.Stdin,
        tx: os.Stdout,
        // MAX_SIZE = 1 MB
        reader: bufio.NewReaderSize(os.NewFile(3, "pipe"), 1024*1024),
        writer: bufio.NewWriterSize(os.NewFile(4, "pipe"), 512*1024),
        rmu:    &sync.Mutex{},
        wmu:    &sync.Mutex{},
    }
    //创建ticker定时器，定时刷新一次缓存（调用 c.Flush() 函数）
    go func() {
        ticker := time.NewTicker(time.Millisecond * 200)
        defer ticker.Stop()
        for {
            <-ticker.C
            if err := c.Flush(); err != nil {
                break
            }
        }
    }()
    return
}

func (c *Client) Flush() (err error) {
	c.wmu.Lock()
	defer c.wmu.Unlock()
	if c.writer.Buffered() != 0 {
        // 发送数据
		err = c.writer.Flush()
	}
	return
}
```

接下来是`SendRecord`与`ReceiveTask`方法：

1.  `ReceiveTask`：从上行`os.Pipe`获取数据，并格式化生成`Task`类型的数据
2.  `SendRecord`：plugin将要采集的数据通过该方法写到`os.Pipe`的写端，即`Client.writer`

```GO
func (c *Client) SendRecord(rec *Record) (err error) {
	c.wmu.Lock()
	defer c.wmu.Unlock()
	size := rec.Size()
	err = binary.Write(c.writer, binary.LittleEndian, uint32(size))
	if err != nil {
		return
	}
	var buf []byte
	buf, err = rec.Marshal()
	if err != nil {
		return
	}
	_, err = c.writer.Write(buf)
	return
}

func (c *Client) ReceiveTask() (t *Task, err error) {
	c.rmu.Lock()
	defer c.rmu.Unlock()
	var len uint32
	err = binary.Read(c.reader, binary.LittleEndian, &len)
	if err != nil {
		return
	}
	var buf []byte
	buf, err = c.reader.Peek(int(len))
	if err != nil {
		return
	}
	_, err = c.reader.Discard(int(len))
	if err != nil {
		return
	}
	t = &Task{}
	err = t.Unmarshal(buf)
	return
}
```


####    transport

##  0x03    AgentServer

####    数据流

####    指令流


##  0x04    参考
-   [最后防线：三款开源 HIDS 功能对比评估](https://mp.weixin.qq.com/s?__biz=MzU4NjY0NTExNA==&mid=2247488176&idx=1&sn=879142f6acbdcb291b432f8d4f6a45aa&chksm=fdf979a5ca8ef0b3b979a8cfe9b27dc8885427eab347fb31c94c2e43c6eac72eae8066652fe7&scene=178&cur_album_id=1783332321308229634#rd)
-   [如何利用 Elkeid 发现生产网内恶意行为](https://www.anquanke.com/post/id/250881)
-   [手把手带你部署 Elkeid 开源项目](https://zhuanlan.zhihu.com/p/498091918)