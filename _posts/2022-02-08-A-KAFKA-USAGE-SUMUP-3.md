---
layout: post
title: 关于 Kafka 应用开发知识点的整理（三）
subtitle: 一些关于kafka客户端库实践经验汇总
date: 2022-02-08
author: pandaychen
header-img:
catalog: true
category: false
tags:
  - Kafka
---

## 0x00 前言
本篇文章，总结下在项目中使用 sarama-kafka 库的一些经验。是对前文[关于 Kafka 应用开发知识点的整理（二）](https://pandaychen.github.io/2022/01/01/A-KAFKA-USAGE-SUMUP-2/)的补充。部分参考阿里云的kafka[最佳实践](https://help.aliyun.com/document_detail/68148.html)

##  0x01    阿里云的最佳实践

####    Producer最佳实践
[Producer最佳实践](https://help.aliyun.com/document_detail/68165.html)，降低发送消息的错误率。文中以JAVA为例（其他语言可参考）：

1、发送消息<br>

发送消息的示例代码如下，时间戳这个可以加，用于在消费端感知消息的时效性（比如重复消费的时候，按照时间过滤掉过期的消息等策略）：
```java
Future<RecordMetadata> metadataFuture = producer.send(new ProducerRecord<String, String>(
        topic,   //消息主题
        null,   //分区编号，建议为null，由Producer分配
        System.currentTimeMillis(),   //时间戳
        String.valueOf(value.hashCode()),   //消息键
        value   //消息值
));
```

####    Consumer最佳实践


##  0x02    客户端的选型
前文[关于 Kafka 应用开发知识点的整理（二）](https://pandaychen.github.io/2022/01/01/A-KAFKA-USAGE-SUMUP-2/)介绍了Shopify/sarama的配置及使用情况，在现网中，主要的客户端有下面几个：

-   [Shopify/sarama](https://github.com/Shopify/sarama)
-   [confluent-kafka-go](https://github.com/confluentinc/confluent-kafka-go/)
-   [segmentio/kafka-go](https://github.com/segmentio/kafka-go)


####    Shopify/sarama
-   优点：完全基于golang实现
-   缺点：问题多

[为什么不推荐使用Sarama Go客户端收发消息？](https://help.aliyun.com/document_detail/266782.html)

####    confluent-kafka-go
此库基于kafka C/C++库`librdkafka`构建，是阿里云官网推荐的kafka客户端，参考下面文档：
-   [Go SDK收发消息](https://www.alibabacloud.com/help/zh/message-queue-for-apache-kafka/latest/sdk-for-go-send-and-consume-messages-by-using-an-ssl-endpoint-with-plain-authentication)
-   [阿里云：kafka-confluent-go-demo](https://github.com/AliwareMQ/aliware-kafka-demos/tree/master/kafka-confluent-go-demo)

-   优点：https://github.com/confluentinc/confluent-kafka-go/#confluents-golang-client-for-apache-kafkatm
-   缺点：编译依赖于CGO，静态编译支持不好，跨平台编译亦如此

此外，sarama提供了mock包

####   segmentio/kafka-go
[segmentio/kafka-go](https://github.com/segmentio/kafka-go)也是一个极佳的备选客户端（许多公司生产环境使用），官方对比其他常见客户端的[优缺点](https://github.com/segmentio/kafka-go#motivations)，也是此库的实现动机。此库完全基于golang实现

-  优点：提供低级API和高级API（如`reader`、`writer`），以writer为例，相对低级api，它是并发safe的，还提供连接保持和重试，无需开发者自己实现，另外writer还支持sync和async写、带context.Context的超时写等
-   缺点：`Writer`的`sync`模式写入比较慢，不推荐使用，推荐使用`async`模式；此外，segmentio/kafka-go没有提供mock测试包，官方推荐：需要自己建立环境测试，在本地启动一个kafka服务，然后运行测试


##  0x03  再看Kafka的高可用：消息备份和同步机制


##  0x04  

Consumer 的可靠性策略
        Consumer的可靠性策略集中在consumer的投递语义上，即

何时消费，消费到什么？
消费是否会丢？
消费是否会重复？
       这些语义场景，可以通过kafka消费者的而部分参数进行配置，简单来说有以下3中场景：

AutoCommit（at most once, commit后挂，实际会丢）
enable.auto.commit = true

auto.commit.interval.ms

       配置如上的consumer收到消息就返回正确给 brocker, 但是如果业务逻辑没有走完中断了，实际上这个消息没有消费成功。这种场景适用于可靠性要求不高的业务。其中auto.commit.interval.ms代表了自动提交的间隔。比如设置为1s提交1次，那么在1s内的故障重启，会从当前消费offset进行重新消费时，1s内未提交但是已经消费的msg, 会被重新消费到。

手动Commit（at least once, commit前挂，就会重复, 重启还会丢）
enable.auto.commit = false

       配置为手动提交的场景下，业务开发者需要在消费消息到消息业务逻辑处理整个流程完成后进行手动提交。如果在流程未处理结束时发生重启，则之前消费到未提交的消息会重新消费到，即消息显然会投递多次。此处应用与业务逻辑明显实现了幂等的场景下使用。

      特别应关注到在golang中sarama库的几个参数的配置，

sarama.offset.initial  （oldest, newest）

offsets.retention.minutes

       intitial = oldest代表消费可以访问到的topic里的最早的消息，大于commit的位置，但是小于HW。同时也受到broker上消息保留时间的影响和位移保留时间的影响。不能保证一定能消费到topic起始位置的消息。

       如果设置为newest则代表访问commit位置的下一条消息。如果发生consumer重启且autocommit没有设置为false, 则之前的消息会发生丢失，再也消费不到了。在业务环境特别不稳定或非持久化consumer实例的场景下，应特别注意。

      一般情况下， offsets.retention.minutes为1440s。

3 Exactly once, 很难，需要msg持久化和commit是原子的

        消息投递且仅投递一次的语义是很难实现的。首先要消费消息并且提交保证不会重复投递，其次提交前要完成整体的业务逻辑关于消息的处理。在kafka本身没有提供此场景语义接口的情况下，这几乎是不可能有效实现的。一般的解决方案，也是进行原子性的消息存储，业务逻辑异步慢慢的从存储中取出消息进行处理。

##  0x04  关于consumer的一些再认知
收集下笔者遇到的一些问题

1、如何监控分区消费堆积（消费过慢）？

2、采用sarama消费，客户端设置为`OffsetNewest`、自动提交模式，那么如果客户端在重启过程中发生了若干分钟的延迟（期间分区有数据写入且正常），那么在重启之后，消费者从何处开始消费？


##  避免rebalance的方法
消费客户端（Consumer）频繁出现Rebalance
心跳超时会引发Rebalance，可以通过参数调整、提高消费速度等方法解决。更多信息，请参见为什么消费客户端频繁出现Rebalance？。

为什么消费客户端频繁出现Rebalance？
更新时间：2023-05-25 10:03:53
产品详情
相关技术圈
我的收藏
可能是云消息队列 Kafka 版客户端版本过低或者Consumer没有独立线程维持心跳。

问题现象
使用云消息队列 Kafka 版时，消费客户端频繁出现Rebalance。

可能原因
可能导致故障的原因包括：

v0.10.2之前版本的客户端：Consumer没有独立线程维持心跳，而是把心跳维持与poll接口耦合在一起。其结果就是，如果用户消费出现卡顿，就会导致Consumer心跳超时，引发Rebalance。

v0.10.2及之后版本的客户端：如果消费时间过慢，超过一定时间（max.poll.interval.ms设置的值，默认5分钟）未进行poll拉取消息，则会导致客户端主动离开队列，而引发Rebalance。

解决方案
首先您需要了解以下几点信息：

session.timeout.ms：心跳超时时间（可以由客户端自行设置）。

max.poll.records：每次poll返回的最大消息数量。

v0.10.2之前版本的客户端：心跳是通过poll接口来实现的，没有内置的独立线程。

v0.10.2及之后版本的客户端：为了防止客户端长时间不进行消费，Kafka客户端在v0.10.2及之后的版本中引入了max.poll.interval.ms配置参数。

参考以下说明调整参数值：

session.timeout.ms：v0.10.2之前的版本可适当提高该参数值，需要大于消费一批数据的时间，但不要超过30s，建议设置为25s；而v0.10.2及其之后的版本，保持默认值10s即可。

max.poll.records：降低该参数值，建议远远小于<单个线程每秒消费的条数> * <消费线程的个数> *<max.poll.interval.ms>的积。

max.poll.interval.ms：该值要大于<max.poll.records> / (<单个线程每秒消费的条数> * <消费线程的个数>)的值。

尽量提高客户端的消费速度，消费逻辑另起线程进行处理。

减少Group订阅Topic的数量，一个Group订阅的Topic最好不要超过5个，建议一个Group只订阅一个Topic。

将客户端升级至0.10.2以上版本。

##  重复消费


##      0x06    一个关于consumer的问题排查经过
在工作中遇到过这样的问题，测试环境消费者进程重启后，不消费重启前一段时间（间隔不久）的数据，大致描述如下：

消费者代码采用``消费位点模式，使用消费者组进行消费，消费者进程的启动时间序列大致如下：

![]()

问题是consumer不消费data的数据？

## 0x05 参考

- [Kafka最佳实践](https://help.aliyun.com/zh/apsaramq-for-kafka/best-practices/)
- [Kafka 设计实现与最佳实践之客户端篇](https://xie.infoq.cn/article/b15e3cf54096172bee0ecace6)
- [Golang 中如何正确的使用 sarama 包操作 Kafka](https://www.cnblogs.com/wishFreedom/p/15131600.html)
- [Producer implementation](https://github.com/Shopify/sarama/wiki/Producer-implementation)
- [Go社区主流Kakfa客户端简要对比](https://tonybai.com/2022/03/28/the-comparison-of-the-go-community-leading-kakfa-clients/)
- [订阅者最佳实践](https://help.aliyun.com/zh/apsaramq-for-kafka/best-practices-for-consumers?spm=a2c4g.11186623.0.i0)
- [CKafka 数据可靠性说明](https://cloud.tencent.com/document/product/597/36186)

转载请注明出处，本文采用 [CC4.0](http://creativecommons.org/licenses/by-nc-nd/4.0/) 协议授权
https://itzones.cn/2020/04/01/Kafka%E6%B6%88%E8%B4%B9%E7%BB%84%E4%BD%8D%E7%A7%BB%E9%87%8D%E8%AE%BE/


https://help.aliyun.com/zh/apsaramq-for-kafka/best-practices-for-consumers?spm=a2c4g.11186623.0.i0

https://help.aliyun.com/zh/apsaramq-for-kafka/why-do-rebalances-frequently-occur-on-my-consumer-client

https://help.aliyun.com/zh/apsaramq-for-kafka/best-practices-for-producers?spm=a2c4g.11186623.0.i0
