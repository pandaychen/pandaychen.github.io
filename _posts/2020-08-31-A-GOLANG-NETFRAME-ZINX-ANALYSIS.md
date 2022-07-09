---
layout:     post
title:      Golang TCP 并发服务器框架：Zinx
subtitle:   一个基于 Golang 轻量级 TCP 并发服务器框架分析
date:       2020-08-31
author:     pandaychen
header-img: img/golang-horse-fly.png
catalog: true
category:   false
tags:
    - Zinx
    - Golang
    - 网络框架
---

##  0x00    前言
zinx的[开发文档](https://aceld.gitbooks.io/zinx/content/)基本上描述的很详细了，对初学者比较友好。笔者先前基于reactor模型实现过一个TCP网络框架[tcpframe](https://github.com/pandaychen/tcpframe)，要点是：
1.  多进程模型
2.  Reactor反应堆模式，基于事件驱动的循环
3.  基于TLV的协议通信，避免粘包

本文就以上面3个维度分析下zinx网络框架的实现的核心思路。

##  0x01    zinx的架构
![architecture](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/zinx/zinx-1.png)


##  0x02    代码分析
先看下Zinx的网络抽象通用定义，包含如下几个方面：
-   连接管理
-   连接定时器管理
-   收/发送Tcp报文处理
-   业务处理包（业务包）逻辑
-   golang的通用网络模型

1、`IServer`：服务接口<br>
```golang
//定义服务接口
type IServer interface {
	Start()		//启动服务器方法
	Stop() 		//停止服务器方法
	Serve() 	//开启业务服务方法
	AddRouter(msgID uint32, router IRouter) //路由功能：给当前服务注册一个路由业务方法，供客户端链接处理使用
	GetConnMgr() IConnManager 				//得到链接管理
	SetOnConnStart(func(IConnection)) 		//设置该Server的连接创建时Hook函数
	SetOnConnStop(func(IConnection)) 		//设置该Server的连接断开时的Hook函数
	CallOnConnStart(conn IConnection) 		//调用连接OnConnStart Hook函数
	CallOnConnStop(conn IConnection) 		//调用连接OnConnStop Hook函数
	Packet() Packet
}
```

2、`IConnection`
连接mod层接口<br>

```golang
//定义连接接口
type IConnection interface {
	Start() 			//启动连接，让当前连接开始工作
	Stop() 				//停止连接，结束当前连接状态M
	Context() context.Context 		//返回ctx，用于用户自定义的go程获取连接退出状态

	GetTCPConnection() *net.TCPConn //从当前连接获取原始的socket TCPConn
	GetConnID() uint32 				//获取当前连接ID
	RemoteAddr() net.Addr 			//获取远程客户端地址信息

	SendMsg(msgID uint32, data []byte) error 		//直接将Message数据发送数据给远程的TCP客户端(无缓冲)
	SendBuffMsg(msgID uint32, data []byte) error	//直接将Message数据发送给远程的TCP客户端(有缓冲)

	SetProperty(key string, value interface{}) 		//设置链接属性
	GetProperty(key string) (interface{}, error)	//获取链接属性
	RemoveProperty(key string) 						//移除链接属性
}
```

3、`IConnManager`：连接管理抽象层<br>
```golang
type IConnManager interface {
	Add(conn IConnection)                   //添加链接
	Remove(conn IConnection)                //删除连接
	Get(connID uint32) (IConnection, error) //利用ConnID获取链接
	Len() int                               //获取当前连接
	ClearConn()                             //删除并停止所有链接
}
```

4、`IRouter`：路由接口<br>
```golang
//路由接口， 这里面路由是 使用框架者给该链接自定的 处理业务方法
//路由里的IRequest 则包含用该链接的链接信息和该链接的请求数据信息
type IRouter interface {
	PreHandle(request IRequest)  //在处理conn业务之前的钩子方法
	Handle(request IRequest)     //处理conn业务的方法
	PostHandle(request IRequest) //处理conn业务之后的钩子方法
}
```

5、`IRequest`<br>
```golang
//实际上是把客户端请求的连接信息 和 请求的数据 包装到了 Request里
type IRequest interface {
	GetConnection() IConnection //获取请求连接信息
	GetData() []byte            //获取请求消息的数据
	GetMsgID() uint32           //获取请求的消息ID
}
```

6、`Packet`接口：<br>
```golang
type Packet interface {
	Unpack(binaryData []byte) (IMessage, error)
	Pack(msg IMessage) ([]byte, error)
	GetHeadLen() uint32
}
```

7、`IMsgHandle`：消息的管理封装，包含了业务处理worker协程池的管理<br>
```golang
//消息管理抽象层
type IMsgHandle interface {
	DoMsgHandler(request IRequest)          //马上以非阻塞方式处理消息
	AddRouter(msgID uint32, router IRouter) //为消息添加具体的处理逻辑
	StartWorkerPool()                       //启动worker工作池
	SendMsgToTaskQueue(request IRequest)    //将消息交给TaskQueue,由worker进行处理
}
```

8、`IMessage`：设置传输消息的通用字段<br>
```golang
//将请求的一个消息封装到message中，定义抽象层接口
type IMessage interface {
	GetDataLen() uint32 //获取消息数据段长度
	GetMsgID() uint32   //获取消息ID
	GetData() []byte    //获取消息内容

	SetMsgID(uint32)   //设置消息ID
	SetData([]byte)    //设置消息内容
	SetDataLen(uint32) //设置消息数据段长度
}
```

9、`IDataPack`：封包数据和拆包数据<br>
```golang
//封包数据和拆包数据
//直接面向TCP连接中的数据流,为传输数据添加头部信息，用于处理TCP粘包问题
type IDataPack interface {
	GetHeadLen() uint32                //获取包头长度方法
	Pack(msg IMessage) ([]byte, error) //封包方法
	Unpack([]byte) (IMessage, error)   //拆包方法
}
```

##  0x03    数据流程
Zinx的主要运行流程如下图所示：
![zinx-module](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/zinx/zinx-module.png)

####	封包/拆包
![tlv](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/zinx/zinx_tcp_tlv.png)


##  0x04    细节&&总结

##  0x05    参考
-   [Zinx project](https://github.com/aceld/zinx/blob/master/znet/server.go)
-   [Zinx--Golang轻量级并发服务器框架](https://aceld.gitbooks.io/zinx/content/)
-   [聊聊 Go Socket 框架 Teleport 的设计思路](https://my.oschina.net/henrylee2cn/blog/2209264)
-   [gnet: 一个轻量级且高性能的 Go 网络框架](https://strikefreedom.top/go-event-loop-networking-library-gnet)