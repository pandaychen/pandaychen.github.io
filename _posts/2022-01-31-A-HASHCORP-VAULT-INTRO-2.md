---
layout:     post
title:      Hashcorp Vault 实战（应用篇）
subtitle:   更安全的 Secret 存储系统：Vault
date:       2022-01-31
author:     pandaychen
header-img: img/super-mario.jpg
catalog: true
category:   false
tags:
    - Vault
    - 网络安全
---

## 0x00 前言

前文[Hashcorp Vault 使用](https://pandaychen.github.io/2020/01/03/A-HASHCORP-VAULT-INTRO/)介绍了Vault的基础知识，本文基于最近的实践小结下使用vault的一些问题及解决。

1.  如何部署高可用的vault集群（后端选择MySQL）？
2.  如何优雅的重启vault
3.  vault-API调用


##  0x01    vault的高可用模式
官方文档提供了HA的[部署方案](https://www.vaultproject.io/docs/concepts/ha)，有两个地方都需要考虑：
-   服务的高可用
-   存储的高可用

####  vault的HA机制

Vault 支持多服务器部署模式（运行多个 Vault 服务器）以实现高可用性。使用支持高可用的数据存储时，会自动启用高可用模式。可以通过启动服务器并查看输出数据存储信息之后是否紧跟着输出`HA available`来判断数据存储是否支持高可用性模式；如果是的话，则 Vault 将自动使用 HA 模式。

为了获得高可用性，其中某个 Vault 服务器节点会在数据存储中成功获取锁。获取到锁的服务器节点将成为主节点；所有其他节点成为备节点。此时，如果备节点收到请求，它们将根据集群的当前配置和状态进行请求转发或客户端重定向。


####  服务的高可用
一般现网中通常采用两种高可用的部署方式：
1.  前面挂CLB负载均衡
2.  Vault 本身的active-standby模式

官方给出了如下文档，供参考：
-   基于LB的部署，[文档](https://www.vaultproject.io/docs/concepts/ha#behind-load-balancers)



####  存储的高可用

######   使用Mysql作为后端存储
我们项目中是采用MySQL作为后端存储的，[官方文档](https://www.vaultproject.io/docs/configuration/storage/mysql#mysql-examples)：

```text
The MySQL storage backend is used to persist Vault's data in a MySQL server or cluster.

High Availability – the MySQL storage backend supports high availability. Note that due to the way mysql locking functions work they are lost if a connection dies. If you would like to not have frequent changes in your elected leader you can increase interactive_timeout and wait_timeout MySQL config to much higher than default which is set at 8 hours.

Community Supported – the MySQL storage backend is supported by the community. While it has undergone review by HashiCorp employees, they may not be as knowledgeable about the technology. If you encounter problems with them, you may be referred to the original author.
```

推荐的Mysql后端存储配置如下，其中关于高可用的参数说明在[此](https://www.vaultproject.io/docs/configuration/storage/mysql#mysql-examples)
```text
ui = true

# HTTPS listener
listener "tcp" {
  address       = "1.1.1.1:8200"
  tls_disable      = "true"
  tls_cert_file = "/data/vault/tls/tls.crt"
  tls_key_file  = "/data/vault/tls/tls.key"
}

storage "mysql" {
  ha_enabled = "true"
  address = "xxxx:3306"
  username = "test"
  password = "xxxx"
  database = "vault"
  max_parallel = 256
  max_idle_connections = 64
  max_connection_lifetime = 1024
  plaintext_connection_allowed = "true"
}
 
api_addr = "http://1.1.1.1:8200"
cluster_addr = "http://1.1.1.1:8201"
```


####  官方推荐的HA部署
官方给了几种高可用集群的组件文档，这里简单罗列一下：

1、[Vault High Availability with Consul](https://learn.hashicorp.com/tutorials/vault/ha-with-consul?in=vault/day-one-consul)：后端使用consul集群的active-standby模式<br>

![vault-consul](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/vault/vault-ha-consul.png)

下图是具体的部署架构：
![vault-consul](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/vault/vault-ha-consul-1.png)


2、Vault-LB 与active-standby模式<br>

![vault-mysql](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/vault/vault-rds.png)

3、基于AzureRM 的[部署方案](https://github.com/hashicorp/terraform-azurerm-vault)<br>

![vault-azure](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/vault/vault-azure.png)

4、使用DynamoDB做后端的[部署模式](https://github.com/avantoss/vault-infra)<br>
![vault-dynamodb](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/vault/vault-DynamoDB.png)

##  0x02    Secrets Engines
Secrets Engines的本质是什么？如kv就是常用的一种Secrets Engine。Secret Engine 就是体现了Vault系统的可插拔性。它允许在保持 Vault 核心稳定的前提下，支持多种不同的加密渠道，每种渠道有各自的优点和适用范围，同时能够通过统一的接口进行管理，默认Vault集群有如下的Engine：

```bash
[root@VM_120_245_centos ~]# vault secrets list
Path          Type         Description
----          ----         -----------
cubbyhole/    cubbyhole    per-token private secret storage
identity/     identity     identity store
secret/       kv           key/value secret storage
sys/          system       system endpoints used for control, policy and debugging
define/your/own/path/    kv           kv_97d4cb4c           n/a
```

从上面的输出可以看到有`2`个类型为 kv 的Engine，分别加载到了 `secret/`、`define/your/own/path/` 的路径上，其描述为**键值对加密存储**。之所以能够向指定的路径进行读写，是因为有 Secret Engine 的支持，没有加载的路径是不能访问的。KV 是最简单的engine（默认加载），它允许在我们存储中以键值对的形式保存私密数据。


####    kv
关于kv，有几个要点：
1.  可以使用`enable`指令，将 kv 加载到其他位置，如`vault secrets enable -path=other/path kv`，同一个引擎engine可以被加载到不同的路径下（可以想象为同一个 engine 类的多个实例），每个路径下的数据都是彼此独立的，它们之间不会互相干扰
2.  kv有两个[版本](https://www.vaultproject.io/docs/secrets/kv)：`Version 1`和`Version 2`，区别是`Version 2`提供了键值对的版本化存储，可以保存一定数量的历史版本（类似于Etcd的MVCC）。通过下面的指令，将v1升级至V2，参考下面的例子
3.  同一路径不能被重复创建


```bash
[root@VM_120_245_centos ~]# vault secrets enable -path=define/your/own/path kv
Success! Enabled the kv secrets engine at: define/your/own/path/

#已存在的路径升级至V2版本
[root@VM_120_245_centos ~]#  vault kv enable-versioning define/your/own/path/ 
Success! Tuned the secrets engine at: define/your/own/path/

[root@VM_120_245_centos ~]# vault secrets enable -path=other/path kv
Success! Enabled the kv secrets engine at: other/path/

[root@VM_120_245_centos ~]# vault secrets enable -path=other/ kv
Error enabling: Error making API request.

URL: POST http://127.0.0.1:8200/v1/sys/mounts/other
Code: 400. Errors:

* path is already in use at other/path/
```

1、读/写入v2版本的数据<br>
这里有个细节需要注意，在v2版本，使用vault命令行操作的路径是`define/your/own/path/test1`，但是实际的存储路径是`define/data/your/own/path/test1`，不过后面这个路径不能直接操作，在API调用时需要格外注意：

```bash
[root@VM_120_245_centos ~]# vault kv put define/your/own/path/test1 key1=value1
Key              Value
---              -----
created_time     2022-05-27T17:17:25.278524978Z
deletion_time    n/a
destroyed        false
version          1

[root@VM_120_245_centos ~]# vault kv get define/your/own/path/test1 
====== Metadata ======
Key              Value
---              -----
created_time     2022-05-27T17:17:25.278524978Z
deletion_time    n/a
destroyed        false
version          1

==== Data ====
Key     Value
---     -----
key1    value1
```

2、获取metadata数据<br>
```bash
[root@VM_120_245_centos ~]# vault kv metadata get define/your/own/path/test1 
========== Metadata ==========
Key                     Value
---                     -----
cas_required            false
created_time            2022-05-27T17:17:25.278524978Z
current_version         1
delete_version_after    0s
max_versions            0
oldest_version          0
updated_time            2022-05-27T17:17:25.278524978Z

====== Version 1 ======
Key              Value
---              -----
created_time     2022-05-27T17:17:25.278524978Z
deletion_time    n/a
destroyed        false
```

3、根据version获取指定版本的值<br>

####    小结
1.  对于kv类型的v2版本数据，所有的版本化数据的路径实际上都是包含`data/`前缀的，根路径后面紧跟着就是`data/`，这个特性会影响policy的配置
2.  由于v2的这种特点，建议使用该KV引擎存储机密时，应避免在路径中包含`data`、`metadata`、`delete`、`unddelete`、`destroy`等关键字，以免发生歧义与冲突


##  0x03    自动解封vault

## 0x04 参考

- [Read Health Information](https://www.vaultproject.io/api-docs/system/health)
- [HA vault cluster failing to communicate with MySQL db endpoint from another cluster #8458](https://github.com/hashicorp/vault/issues/8458)
- [Designing High Availability for HashiCorp Vault in AWS](https://www.ahead.com/resources/designing-high-availability-for-hashicorp-vault-in-aws/)
- [Auto Unseal](https://github.com/hashicorp/vault/blob/main/website/content/docs/concepts/seal.mdx#auto-unseal)
- [](https://groups.google.com/g/vault-tool/c/E9wLwBUkYsM)
- [Vault - High Availability and Scalability](https://blogs.halodoc.io/vault-high-availability-and-scalability-2/)