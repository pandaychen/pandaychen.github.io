---
layout: post
title: Cronsun：任务统一集中管理（调度）系统设计与分析
subtitle: 分析一款典型的分布式任务调度系统
date: 2022-06-20
header-img: img/super-mario.jpg
author: pandaychen
catalog: true
tags:
  - Crontab
  - 分布式
---

##  0x00    开篇
cronsun 是一个分布式任务系统，单个节点和 Linux 机器上的 `crontab` 类似，目的是解决多台 Linux 机器上 crontab 任务管理不方便的问题，同时提供任务高可用的支持（当某个节点死机的时候可以自动调度到正常的节点执行），此外，有页面配置及告警邮件支持等。汇总如下3点：

1. 替换 crontab
2. 执行不能单点失败的任务，具备一定可靠性运行
3. 简单的任务调度能力（但cronsun 不是任务队列）


本文分析下其实现的思路，之前已有相关文章：
1.  [Golang CRON 库 Crontab 的使用与设计](https://pandaychen.github.io/2021/10/05/A-GOLANG-CRONTAB-V3-BASIC-INTRO/)
2.  [基于 CRON 库扩展的分布式 Crontab 的实现](https://pandaychen.github.io/2022/01/16/A-GOLANG-CRONTAB-V3-ANALYSIS/)

##	0x01	cronsun架构

```text
                                      [web]
                                        |
                           --------------------------
 (add/del/update/exec jobs)|                        |(query job exec result)
                         [etcd]                 [mongodb]
                           |                        ^
                  --------------------              |
                  |        |         |              |
               [node.1]  [node.2]  [node.n]         |
   (job exec fail)|        |         |              |
[send mail]<-----------------------------------------(job exec result)
```

简化为如下架构：
![img](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/2022/dcrontab/crontab-normal-flow-with-etcd.png)

值得说明的是：
1.	图中的worker节点通常是单机部署，上面部署一个Cron-Node 节点，主要负责调度和执行任务
2.	图中的master节点（即Cron-Web节点）一般是无状态的，容易实现水平扩展，主要负责管理任务、查看任务执行日志

####	可靠性说明
1、worker节点的可靠性<br>
-	cronnode被设计成一个常驻进程，追求稳定和高可用
-	若cronnode和etcd服务的连接中断了
	-	断开连接之前已经下发的任务会正常执行
	-	在断开连接期间新建、修改、删除的任务无法更新到节点
	-	自动和etcd进行重连
	-	和etcd重新连接上后，会重新加载和该节点相关的全部任务，保证正确性；
-	若cronnode和MongoDB数据库的连接中断了
	-	所有任务依然会正常执行
	-	在断开连接期间执行完成的任务，日志会因为无法写入到MongoDB而看不到任务的执行记录和任务日志，不影响任务正常执行
	-	会自动和MongoDB进行重连，重连后执行记录和任务日志会正常写入MongoDB

2、master节点的可靠性<br>
-	若cronweb进程崩溃了
	-	不影响cronnode节点和任务正常执行
	-	报警邮件无法发送
-	若cronweb和etcd服务的连接中断
	-	cronweb无法访问
	-	报警邮件无法发送
-	若cronweb和MongoDB服务的连接中断
	-	cronweb无法访问
	-	报警邮件无法发送
-	可以部署多个cron-web节点，可以访问任一节点正常管理任务和查看日志

笔者阅读过的分布式crontab实现，几乎都是与该架构类似（不同之处是etcd替换为redis或其他一些分布式存储）；
1.  [gocron](https://github.com/ouqiang/gocron)：定时任务管理系统，文档见[此](https://github.com/ouqiang/gocron/wiki)


##  0x02  cronsun配置介绍
本小节来自[官方文档](https://github.com/shunfei/cronsun/wiki/%E4%BB%BB%E5%8A%A1%E9%85%8D%E7%BD%AE)，了解下cronsun支持的任务类型等属性

####  任务类型（任务执行策略机制不同）
- 普通任务：任务会在每一台指定的节点上面执行（不排他）
- 单机单进程任务：任务会在所有指定的节点中选择其中一个节点来执行，并且同时有且只有一个任务进程在跑，直到本次任务执行结束（类似于dcron中的job）
- 组级别普通任务：与单机单进程类型不一样的地方是，**去除了同时有且只有一个任务进程在跑的限制**，只要任务调度时间到，就会在某个节点上面开始另一个任务进程

以上三种任务对应于node节点的异步监听的不同goroutine，代码[如下](https://github.com/shunfei/cronsun/blob/master/node/node.go#L581)：
```golang
// 启动服务
func (n *Node) Run() (err error) {
	go n.keepAlive()

	defer func() {
		if err != nil {
			n.Stop(nil)
		}
	}()

	if err = n.loadJobs(); err != nil {
		return
	}

	n.Cron.Start()
	go n.watchJobs()
	go n.watchExcutingProc()
	go n.watchGroups()    //监控组级别任务
	go n.watchOnce()  
	go n.watchCsctl()
	n.Node.On()
	return
}
```

####  任务分组
任务分组只是出于方便管理的原因而添加的功能，在添加/修改任务时，可以选择已有的分组，或者直接在输入框填写新的分组名称来创建新分组，同时把任务分配给新分组

####  任务脚本
- 脚本：相当于 `crontab` 中的执行命令，复杂的或者需要管道支持的任务需要写成 `shell` 脚本
- 参数：脚本参数以空格作为分割符

####  执行结果
每个任务最终都会有一个结果，cronsun 通过任务的 `exit code` 来判断任务执行是否成功，只有 `exit code` 为 `0` 时被认为任务是执行成功的。任务的结果会影响到一些配置，以及是否需要告警等

####  任务的超时设置
任务执行超过指定时间时将被强行结束，并且视为任务执行失败

####  一个节点上面该任务并行数
在同一时间内，同个节点上面最多可以有多少个相同任务进程一起执行。比如当你的任务执行一次需要 `5` 个小时，但是你设置了间隔 `1` 小时执行一次，那么大部分时间内都会有 `4` 个相同任务进程在执行，如果把此参数设置为 `2`，那么同一时间内只能有最多 `2` 个相同任务进程。

####  重试 & 重试间隔
- 只在任务失败时用到，设置一个大于 `0` 的次数，在任务执行失败时，cronsun 会一直尝试再次执行，直到重试次数达到上限，或者任务执行成功了
- 重试间隔表示任务失败后，等待多久才开始重新下一次尝试

####  日志存储 & 清除
- 存储：目前 cronsun 会直接捕获并合并任务的 `stdout` 和 `stderr` 存储到 MongoDB，此方式只适用于少量的日志，对于有大量日志输出的任务，需另外保存
- 清除：打开配置文件 `web.json`，修改 `LogCleaner.EveryMinute` 为一个大于 `0` 的数，比如 `60`，表示每隔 `60` 分钟定时清除一次过期日志，过期时间是通过 `LogCleaner.ExpirationDays` 来定义，设置 `3`，表示 `3` 天前的日志都是过期的，将会被清除
- 为任务单独设置过期时间：可能有些任务的日志比较重要，你想保存久一点，可以在界面上面为任务单独设置过期时间


####	定时器（类似robfig/cron）
-	`1`个任务至少有关联`1`个定时器，定时器包含了任务执行的时间和节点
-	多个定时器之间共享上面的规则和限制
-	任务节点规则优先级：排除的节点 `>` 指定的节点 `>` 节点分组
-	调度规则：规则与 cron 一样，但是这里的设置有 `6` 段，支持秒级调用

```text
0    1    2    3    4    5
|    |    |    |    |    |
|    |    |    |    |    +------ 星期 (0 - 6，星期天 = 0)
|    |    |    |    +------ 月 (1 - 12)
|    |    |    +-------- 日 (1 - 31)
|    |    +---------- 时 (0 - 23)
|    +------------ 分 (0 - 59)
+-------------- 秒 (0 - 59)
```


|字段名	|允许的值	| 允许的特殊字符| 
| :-----:| :----: | :----:  |
|秒		| 0 - 59| 		* / , -| 
|分	| 0 - 59	| * / , -| 
|时	| 0 - 23	| * / , -| 
|日	| 1 - 31	| * / , - ?| 
|月	| 1 - 12 或 JAN-DEC	| * / , -| 
|星期	| 0 - 6 或 SUN-SAT	| * / , - ?| 


特殊符号说明：

-	*：星号会匹配字段中的所有值，如在小时字段用了星号，表示每个小时都会匹配；
-	/： 匹配指定的数字，如 */3 在小时字段中等于 0,3,6,9,12,15,18,21 等被 3 整除的数；
-	,： 匹配分开的值，如 1,3,4,7,8 在小时字段中表示这里面的小时会匹配；
-	-： 匹配范围，例如：1-6，意思等同于 1,2,3,4,5,6；
-	?： 在 日 或者 星期 字段可代替 *；

预定格式如下：

|预定项	|描述	|等价规则|
| :-----:| :----: | :----:  |
|@yearly (或 @annually)	|每年执行一次, 每年 1 月 1 日 0 点开头执行	|0 0 0 1 1 *|
|@monthly	|每月执行一次, 每月 1 日 0 点开头执行	|0 0 0 1 * *|
|@weekly	|每周执行一次, 星期天 0 点开头执行	|0 0 0 * * 0|
|@daily (或 @midnight)	|每天执行一次, 每天 0 点开头执行	|0 0 0 * * *|
|@hourly	|每小时执行一次, 每小时 0 分开头执行	|0 0 * * * *|


##	0x02	可用性介绍




##	0x03	cronsun架构

####	整体架构
cronsun的整体架构如下：
![cronsun-architecture](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/2022/dcrontab/cronsun/cronsun-architecture.png)


####	worker工作流
![flow](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/2022/dcrontab/cronsun/cronsun-task-flow.png)

##	0x04	组件介绍
-	MongoDB
-	Etcd

##	核心代码分析


##	0x05  总结
本项目


##  0x06	参考
-	[分布式任务系统 cronsun](http://bos.itdks.com/786da33844604637be5479c3a16af11e.pdf)
- [还在用crontab? 分布式定时任务了解一下](https://www.cnblogs.com/kevinwan/p/14497753.html)
- [分布式crontab架构](https://www.cnblogs.com/aganippe/p/16012588.html)
- [设计一个分布式定时任务系统](https://yeqown.xyz/2022/01/27/设计一个分布式定时任务系统/)
- [替代crontab，统一定时任务管理系统cronsun简介](https://cloud.tencent.com/developer/article/1072323)