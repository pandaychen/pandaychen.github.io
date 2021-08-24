---
layout:     post
title:      安全：KMS 的那些事
subtitle:   KMS 介绍与使用
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



##  0x01    KMS 的对称加解密

####    信封加密
信封加密（Envelope Encryption）是类似数字信封技术的一种加密手段。这种技术将加密数据的数据密钥封入信封中存储 / 传递和使用，不再使用主密钥直接加解密数据。

（信封加密）的加密流程如下：
1、首先，通过 KMS 创建一个客户主密钥，也就是 CMK<br>
2、CMK（Custom Master Key） 生成后，使用 CMK 生成数据密钥，用户能够得到一个明文数据密钥和一个密文的数据密钥 <br>
3、然后使用明文数据密钥加密明文数据，生成密文 <br>
4、将密文和密文数据密钥一同存储到持久化存储设备或服务中 <br>
5、完成加密后，将明文数据，以及明文数据密钥删除 <br>

当完成上述加密操作后，只保留密文密钥和密文文件，这样即使密文密钥和密文文件被窃取，也无法解密获得用户的明文文件

![kms-Envelope-en.png](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/kms/kms-Envelope-en.png)


（信封加密）的解密流程如下：
1、首先，从上述持久化存储设备或服务中读取密文数据密钥和密文数据 <br>
2、然后，调用 KMS 服务的 `Decrypt` 接口，使用 CMK 解密数据密钥，取得明文数据密钥 <br>
3、最后，使用上一步得到的明文数据密钥解密文件 <br>

![kms-Envelope-en.png](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/kms/kms-Envelope-dec.png)

在上面流程中需要特别注意的是在加密过程中业务系统对明文密钥的处理，由于信封加密场景中采用的是对称加密，故明文密钥不可落盘，需在业务流程的内存中使用，使用完需要立即销毁，否则一旦明文秘钥泄漏，信封加密的安全性就失效了。

另外，后台系统中对数据密钥的处理，可根据业务需求复用同一个数据密钥或针对不同用户、不同时间使用不同的数据密钥进行加密，避免 DEK 秘钥重复导致的安全隐患

加深下印象，这里使用 RSA 和 AES 再次描述下 Envelope Encryption 的逻辑：

1、加密过程 <br>
-   用户创建 CMK RSA 秘钥（保存在 KMS 中）
-   生成 AES 明文密钥
-   使用 AES 明文密钥加密原始数据，得到加密数据
-   使用 RSA 公钥加密 AES 明文密钥，得到密文密钥
-   将密文密钥与加密数据一并存储于信封中

2、解密过程 <br>
-   数据接收者接收到信封，里面包含密文数据与密文秘钥
-   使用 RSA 私钥解密 AES 密文密钥，得到明文密钥
-   使用 AES 明文密钥解密密文数据


##  0x02    参考
-   [AWS Key Management Service](https://docs.aws.amazon.com/zh_cn/kms/latest/developerguide/concepts.html)
-   [腾讯云 KMS](https://cloud.tencent.com/document/product/573/8790)
-   [阿里云 KMS](https://help.aliyun.com/document_detail/42339.html)
-   [AWS CLOUD KMS Data encryption using AWS KMS Key](https://tech.david-cheong.com/data-encryption-using-aws-kms-key/)