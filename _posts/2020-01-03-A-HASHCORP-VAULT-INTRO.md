---
layout:     post
title:      Hashcorp Vault 使用
subtitle:   更安全的 Secret 存储系统：vault
date:       2020-01-03
author:     pandaychen
header-img: img/super-mario.jpg
catalog: true
category:   false
tags:
    - Vault
    - 网络安全
---


##  0x00    前言
我们在工作中是如何管理大量的 Secret 信息的（比如笔者的项目中会涉及到对 OpenSSH 的秘钥及口令存储、以及对此的定时轮转及外部调用）？
-   以配置文件的形式固化，存放于服务器文件或者 Database 中
-   以代码的方式存在于 `git` 私有仓库上，并严格控制此库的访问权限
-   以 KMS（Key Management Service，云服务居多）方式托管在公有云服务上
<br>
我们需要实现的通用密码仓库需要满足如下特性：
-   密文存储（兼顾复杂性）
-   通用性，减少人工修改 secret 的工作量，即提供 RestfulAPI
-   权限控制


本文要介绍 [Vault](https://learn.hashicorp.com/vault) 这款开源的 Secret 管理工具（口令、token、私钥及证书等等），是管理代码中口令、秘钥（防止明文泄漏）极佳的应用实践。此外，KMS（Key Management Service，云服务居多）也是较好的 Secret 管理实践。源码 [在此](https://github.com/hashicorp/vault)

针对此类产品，需要着重关注以下几点：
1.  Secret 的存储方式，支持的存储后端
2.  Secret 的加密方式及算法
3.  系统的权限控制、权限分配（哪些人 / 客户端能访问哪些机器的 Secret）
4.  系统的认证方式，客户端使用什么方式访问 RestfulAPI
5.  系统的 Secret 存储方式及过期机制
6.  系统的高可用性如何保证
7.  系统对外接口的 QPS 及并发性能

##  0x01    Vault 基本原理
vault 的基础应用场景如下：
![vault-basic](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/vault/vault-basic.png)

Vault 的使用场景一般为：
1.  系统用户（如运维同事）通过 `HTTP-Vault-API`、Vault 命令行工具等将 secret data 写入 Vault
2.  Vault 再将加密的数据存储到后端
3.  外部用户（如开发人员，各类脚本或者应用程序）通过 `HTTP-Vault-API`、Vault 命令行工具等方式来获取到仅仅与自己账号相关联的 secret data，这里就涉及到 Valut 的权限细粒度管理


vault 的架构如下：
![vault-architecture](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/vault/vault-%E6%9E%B6%E6%9E%84%E5%9B%BE.png)

####    一些关于 Vault 的名词
1、Token<br>
[Token](https://learn.hashicorp.com/tutorials/vault/tokens)


####    Shamir 密钥分享算法（shamir secret sharing）
Vault 中给出了 Shamir 算法的 [实现](https://github.com/hashicorp/vault/blob/master/shamir/shamir.go)，该密钥分享算法的基本思想是分发者通过秘密多项式，将秘密 secret 分解为 `n` 个秘密持有者，其中任意至少于 `k` 个秘密均能恢复密文，即某一个秘密通常不能由单个持有者保存，必须将秘密分由多人保管并且只有当多人同时在场时秘密才能得以恢复。

![vault-key-share](https://github.com/pandaychen/pandaychen.github.io/blob/master/blog_img/vault/vault-shamir-key-algorithm.png)

####    存储
Vault 支持多种存储后端：具体 [在此](https://github.com/hashicorp/vault/tree/master/plugins/database)，生产架构的后端存储使用 `HA` 方式进行部署，比如 consul/etcd/mysql 集群等。

-   [mysql](https://github.com/hashicorp/vault/blob/master/plugins/database/mysql/mysql.go)

##  基本功能使用
1、配置 Mysql 作为存储后端，启动 vault<br>
```bash
[root@VM_120_245_centos ~/vault]# cat vault.hcl
disable_mlock  = true
ui=true
storage "mysql" {
    address = "9.134.120.245:3306"
    username = "root"
    password = "a@A12345"
    database = "vault"
    table = "vault"
}
listener "tcp" {
 address     = "127.0.0.1:8200"
 tls_disable = 1
}

[root@VM_120_245_centos ~/vault]# vault server -config=vault.hcl
```

2、初始化 vault，得到 `5` 个子秘钥及 `root Token`
```bash
[root@VM_120_245_centos ~/vault]# vault operator init
2021-08-20T21:34:41.973+0800 [INFO]  core: security barrier not initialized
2021-08-20T21:34:42.002+0800 [INFO]  core: security barrier initialized: stored=1 shares=5 threshold=3
2021-08-20T21:34:42.031+0800 [INFO]  core: post-unseal setup starting
2021-08-20T21:34:42.060+0800 [INFO]  core: loaded wrapping token key
2021-08-20T21:34:42.060+0800 [INFO]  core: successfully setup plugin catalog: plugin-directory=""
2021-08-20T21:34:42.060+0800 [INFO]  core: no mounts; adding default mount table
2021-08-20T21:34:42.080+0800 [INFO]  core: successfully mounted backend: type=cubbyhole path=cubbyhole/
2021-08-20T21:34:42.081+0800 [INFO]  core: successfully mounted backend: type=system path=sys/
2021-08-20T21:34:42.081+0800 [INFO]  core: successfully mounted backend: type=identity path=identity/
2021-08-20T21:34:42.136+0800 [INFO]  core: successfully enabled credential backend: type=token path=token/
2021-08-20T21:34:42.137+0800 [INFO]  rollback: starting rollback manager
2021-08-20T21:34:42.137+0800 [INFO]  core: restoring leases
2021-08-20T21:34:42.138+0800 [INFO]  expiration: lease restore complete
2021-08-20T21:34:42.163+0800 [INFO]  identity: entities restored
2021-08-20T21:34:42.164+0800 [INFO]  identity: groups restored
2021-08-20T21:34:42.165+0800 [INFO]  core: usage gauge collection is disabled
2021-08-20T21:34:42.174+0800 [INFO]  core: post-unseal setup complete
2021-08-20T21:34:42.200+0800 [INFO]  core: root token generated
2021-08-20T21:34:42.200+0800 [INFO]  core: pre-seal teardown starting
2021-08-20T21:34:42.200+0800 [INFO]  rollback: stopping rollback manager
2021-08-20T21:34:42.200+0800 [INFO]  core: pre-seal teardown complete
Unseal Key 1: xxx1
Unseal Key 2: xxx2
Unseal Key 3: xxx3
Unseal Key 4: xxx4
Unseal Key 5: xxx5

Initial Root Token: xxx-root-token
```

3、解封 vault，查看解封状态 <br>
```bash
[root@VM_120_245_centos ~/vault]# vault operator unseal xxx1
[root@VM_120_245_centos ~/vault]# vault operator unseal xxx2
[root@VM_120_245_centos ~/vault]# vault operator unseal xxx3

[root@VM_120_245_centos ~/vault]# vault status
Key                Value
---                -----
Seal Type          shamir
Initialized        true
Sealed             true
Total Shares       5
Threshold          3
Unseal Progress    2/3
Unseal Nonce       95ba53e7-63e2-c998-e1b8-4df4bba20ea3
Version            1.8.1
Storage Type       mysql
HA Enabled         false
```

4、使用 `root Token` 登录（首次）<br>
```bash
[root@VM_120_245_centos ~/vault]# vault login root-token
Success! You are now authenticated. The token information displayed below
is already stored in the token helper. You do NOT need to run "vault login"
again. Future Vault requests will automatically use this token.

Key                  Value
---                  -----
token                xxx
token_accessor       xxx
token_duration       ∞
token_renewable      false
token_policies       ["root"]
identity_policies    []
policies             ["root"]
```

5、开启 vault 组件，测试写 / 读 / token 生成等操作 <br>
```bash
[root@VM_120_245_centos ~/vault]# vault secrets enable kv
2021-08-20T21:44:37.837+0800 [INFO]  core: successful mount: namespace="" path=kv/ type=kv
[root@VM_120_245_centos ~/vault]# vault kv put kv/test api_key=abc1234 api_secret=1a2b3c4d^C
[root@VM_120_245_centos ~/vault]# vault kv get kv/test
======= Data =======
Key           Value
---           -----
api_key       abc1234
api_secret    1a2b3c4d
[root@VM_120_245_centos ~/vault]# vault token create -ttl 1h
Key                  Value
---                  -----
token                s.6sfhKw2J2dFfNSMdIyWQGqiS
token_accessor       o8MjM5SuLewdJk0WSZ0oPPrF
token_duration       1h
token_renewable      true
token_policies       ["root"]
identity_policies    []
policies             ["root"]

[root@VM_120_245_centos ~/vault]# export VAULT_TOKEN=s.6sfhKw2J2dFfNSMdIyWQGqiS
[root@VM_120_245_centos ~/vault]# vault kv get kv/test
======= Data =======
Key           Value
---           -----
api_key       abc1234
api_secret    1a2b3c4d
```

6、根据策略模板创建 token<br>
创建 `kv/test` 路径下只读的策略 `test-read-policy`：
```bash
[root@VM_120_245_centos ~/vault]# cat limit-token.hcl
path "kv/test"{
capabilities = ["read"]
}
[root@VM_120_245_centos ~/vault]# vault policy write test-read-policy ./limit-token.hcl
Success! Uploaded policy: test-read-policy
```

根据策略创建 token:
```bash
[root@VM_120_245_centos ~/vault]# vault token create -policy=test-read-policy
Key                  Value
---                  -----
token                s.NMD47aWmzSUWC1bAqalQCWYw
token_accessor       gi6WVfxwJnGqMXhDVoJXI5AU
token_duration       768h
token_renewable      true
token_policies       ["default" "test-read-policy"]
identity_policies    []
policies             ["default" "test-read-policy"]
```

测试 token 的操作情况，只读，写报错，符合既定策略：
```bash
[root@VM_120_245_centos ~/vault]# export VAULT_TOKEN=s.NMD47aWmzSUWC1bAqalQCWYw
[root@VM_120_245_centos ~/vault]# vault kv get kv/test
======= Data =======
Key           Value
---           -----
api_key       abc1234
api_secret    1a2b3c4d5e6f
[root@VM_120_245_centos ~/vault]# vault kv put  kv/test api_key=foo api_secret=bar
Error writing data to kv/test: Error making API request.

URL: PUT http://127.0.0.1:8200/v1/kv/test
Code: 403. Errors:

* 1 error occurred:
        * permission denied
```



##  参考
-   [vault - Documentation](https://www.vaultproject.io/docs)
-   [Secret sharing](https://en.wikipedia.org/wiki/Secret_sharing)
-   [密钥分享 Secret Sharing 介绍](https://zhuanlan.zhihu.com/p/44999983)
-   [小马哥 Devops](https://my.oschina.net/u/3952901?tab=newest&catalogId=7038751)
-   [vault-architecture](https://www.vaultproject.io/docs/internals/architecture)