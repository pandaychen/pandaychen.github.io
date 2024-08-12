---
layout:     post
title:      安全：KMS 的那些事
subtitle:   KMS 原理与使用
date:       2021-07-10
author:     pandaychen
catalog:    true
header-img: img/panda-md-pic7.jpg
tags:
  - KMS
  - 安全
---


##  0x00    前言
密钥管理系统（Key Management Service，KMS）是一款安全管理类服务，可以让您轻松创建和管理密钥，保护密钥的保密性、完整性和可用性，满足用户多应用多业务的密钥管理需求，符合监管和合规要求。

KMS 是基于硬件加密机的云上密钥管理系统，核心服务如下：
- 密钥的全生命周期管理
- 加密、解密算法
- 真随机数
- 密钥轮换

##  0x01    KMS 的加解密

####  敏感数据加密
此场景保护小型敏感数据（小于 4KB），如密钥、证书、配置文件等，此方式直接使用 `CMK` 加密敏感数据信息。`CMK` 通过经过第三方认证的硬件密码模块（HSM）加密保护，从而保障敏感数据的安全性。对用户而言，只需要关注 `Encrypt`/`Decrypt` 接口即可，由 KMS 提供加密计算。

为何限制是小型数据？由于此加密是非对称加密，加密数据长度受密钥长度限制。


![IMG](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/kms/small_data_kms.png)

####    信封加密
一般非对称加密相对对称加密的性能是较差的，那么如何解决加密性能及大数据量加密的问题？KMS 引入了信封加密机制。

信封加密（Envelope Encryption）是类似数字信封技术的一种加密手段。这种技术将加密数据的数据密钥封入信封中存储 / 传递和使用，不再使用主密钥 `CMK` 直接加解密数据。是一种应对海量数据的高性能加解密方案。对于较大的文件或者对性能敏感的数据加密，可以调用类似 API 接口，如 [GenerateDataKey 接口](https://cloud.tencent.com/document/product/573/34419) 生成数据加密密钥 `DEK`，只需要传输数据加密密钥 `DEK` 到 KMS 服务端（必须通过 `CMK` 进行加解密），所有的业务数据都是采用高效的本地对称加密处理，对业务的访问体验影响很小。对用户而言，需要关注通过 `GenerateDataKey` 创建 `DEK`，通过 `Decrypt` 解密 `DEK`。

在对数据加密性能要求较高，数据加密量大的场景下，可通过本地内存中生成 `DEK` 来对本地数据进行加解密，保证了业务加密性能的要求，同时也由 KMS 确保了数据密钥的随机性和安全性。

上面两种方式基本一样，差别在于 `DEK` 的生成方式，是调用云 API 生成，还是在本地内存中生成。

![IMG](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/kms/normal_data_kms.png)

信封加密的加密流程如下：<br>
1、首先，通过 KMS 创建一个客户主密钥，也就是 `CMK`<br>
2、`CMK`（Custom Master Key） 生成后，使用 `CMK` 生成数据密钥，用户能够得到一个明文数据密钥和一个密文的数据密钥 <br>
3、然后使用明文数据密钥加密明文数据，生成密文 <br>
4、将密文和密文数据密钥一同存储到持久化存储设备或服务中 <br>
5、完成加密后，将明文数据，以及明文数据密钥删除 <br>

当完成上述加密操作后，只保留密文密钥和密文文件，这样即使密文密钥和密文文件被窃取，也无法解密获得用户的明文文件。`CMK` 是授权用户在云密钥管理服务中创建的主密钥（不能够取出，只能使用其加 / 解密获取结果），`CMK` 主要用于加密保护数据密钥，产生信封，也可直接用于加密少量的数据

![kms-Envelope-en.png](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/kms/kms_Envelope_en.png)

信封加密的解密流程如下：<br>
1、首先，从上述持久化存储设备或服务中读取密文数据密钥和密文数据 <br>
2、然后，调用 KMS 服务的 `Decrypt` 接口，使用 CMK 解密数据密钥，取得明文数据密钥 <br>
3、最后，使用上一步得到的明文数据密钥解密文件 <br>

![kms-Envelope-en.png](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/kms/kms_Envelope_dec.png)

在上面流程中需要特别注意的是 <font color="#dd0000"> 在加密过程中业务系统对明文密钥的处理，由于信封加密场景中采用的是对称加密，故明文密钥不可落盘，需在业务流程的内存中使用，使用完需要立即销毁，否则一旦明文秘钥泄漏，信封加密的安全性就失效了 </font>。

另外，后台系统中对数据密钥的处理，可根据业务需求复用同一个数据密钥或针对不同用户、不同时间使用不同的数据密钥进行加密，避免 `DEK` 秘钥重复导致的安全隐患

####    信封加密的实例
本小节，使用 RSA 和 AES 加密算法描述下 Envelope Encryption 的逻辑：

1、加密过程 <br>
-   用户创建 `CMK` RSA 秘钥（保存在 KMS 中）
-   生成 AES 明文密钥
-   使用 AES 明文密钥加密原始数据，得到加密数据
-   使用 RSA 公钥加密 AES 明文密钥，得到密文密钥
-   将密文密钥与加密数据一并存储于信封中

2、解密过程 <br>
-   数据接收者接收到信封，里面包含密文数据与密文秘钥
-   使用 RSA 私钥解密 AES 密文密钥，得到明文密钥
-   使用 AES 明文密钥解密密文数据


##  0x02  KMS 密钥分级体系
KMS 密钥管理采用分级管理体系， 如下图所示：

![img](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/kms/kms_mkey_duo_level.png)

1、HSM Key<br>
即上图中的根密钥，由加密机初始化生成的 `HSMKey`，无法导出，只能在加密机内部使用，用于加密保护 `DomainKey`

2、DomainKey<br>
`DomainKey` 即区域密钥，由加密机创建并存储于加密机内，无法导出，只能在加密机使用，用于加密保护 CMK。每个 region 会有多个不同 `DomainKey` 进行轮换加密

3、CMK（Customer Muster Key）<br>
即用户主密钥，`CMK` 对用户来说属于用户的根密钥。 `CMK` 无法导出明文，只能在加密机内部实现加解密。CMK 用于进行数据加解密和 `DEK` 的派生

4、DEK（Data Encrypt Key）<br>
即数据密钥，数据密钥由 `CMK` 加密保护，KMS 对 `DEK` 不做存储，由应用方 / 用户使用管理，这里 `DEK` 的安全性需要由用户自己确保。


##  0x03 一些思考

####  DEK 的安全性及效率
KMS 的信封加密，需要自己保存管理 `DEK`。如何处理 `DEK` 的管理、缓存和容灾等问题？可以使用 hashicorp 的 Vault 来存储 `DEK`。`DEK` 使用的基本流程应该按如下步骤：

1.  `DEK` 初始化通过 KMS `Decrypt` 解密得到明文密钥
2.  明文密钥保留在内存中使用（或者存储在 Vault 集群中），避免每次都调用 KMS 接口
3.  用户通过 `DEK` 明文本地加解密
4.  当不再需要 `DEK` 进行加解密时，清除内存中的密钥，确保密钥明文不落盘


##  0x04    参考
-   [AWS Key Management Service](https://docs.aws.amazon.com/zh_cn/kms/latest/developerguide/concepts.html)
-   [腾讯云 KMS](https://cloud.tencent.com/document/product/573/8790)
-   [阿里云 KMS](https://help.aliyun.com/document_detail/42339.html)
-   [AWS CLOUD KMS Data encryption using AWS KMS Key](https://tech.david-cheong.com/data-encryption-using-aws-kms-key/)
-   [KMS开放API及最佳实践多语言样例](https://github.com/aliyun/alibabacloud-kms-demo)