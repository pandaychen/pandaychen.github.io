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


## 0x0 参考

- [Kafka 设计实现与最佳实践之客户端篇](https://xie.infoq.cn/article/b15e3cf54096172bee0ecace6)
- [Golang 中如何正确的使用 sarama 包操作 Kafka](https://www.cnblogs.com/wishFreedom/p/15131600.html)
- [Producer implementation](https://github.com/Shopify/sarama/wiki/Producer-implementation)
-   [Go社区主流Kakfa客户端简要对比](https://tonybai.com/2022/03/28/the-comparison-of-the-go-community-leading-kakfa-clients/)

转载请注明出处，本文采用 [CC4.0](http://creativecommons.org/licenses/by-nc-nd/4.0/) 协议授权
