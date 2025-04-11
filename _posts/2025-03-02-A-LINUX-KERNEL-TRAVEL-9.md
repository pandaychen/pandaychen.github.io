---
layout:     post
title:  Linux 内核之旅（九）：OverlayFS
subtitle:   
date:       2025-03-02
author:     pandaychen
header-img:
catalog: true
tags:
    - Linux
    - Kernel
---

##  0x00    前言

##  0x01    overlayFS 基础
OverlayFS是一种堆叠文件系统，它依赖并建立在其它文件系统（如`ext4fs`/`xfs`等）之上，并不直接参与磁盘空间结构的划分，仅仅将原来底层文件系统中不同的目录进行（策略式）合并，然后向用户呈现。对于用户来说，它所见到的overlay文件系统根目录下的内容就来自挂载时所指定的不同目录按策略合并之后的结果，挂载文件的基本命令如下：

```BASH
#其中lower1:lower2:lower3表示不同的lower层目录
#不同的目录使用:分隔，层次关系依次为lower1 > lower2 > lower3
mount -t overlay overlay -o lowerdir=lower1:lower2:lower3,upperdir=upper,workdir=work merged
```

上面的指令对应如下的挂载关系，正常执行以上命令后，overlayFS就成功挂载到`merged`目录

![ex1]()

####    分层结构
OverlayFS 将文件系统分为只读层（lowerdir）和可写层（upperdir），通过联合挂载呈现统一视图：

-   lowerdir：只读层，不能被修改。lower 层支持多个目录，优先级依次降低
-   upperdir：可读写层，对文件的创建、修改、删除操作都在这一层体现
-   mergeddir 是 lowerdir 和 upperdir 里的所有文件合并后的挂载目录，是用户最终看到的目录
-   lowerdir 和 upperdir 目录存在同名文件时，lowerdir 的文件将会被隐藏，用户只能看到 upperdir 的文件
-   lowerdir 低优先级的同目录同名文件将会被隐藏
-   如果存在同名目录，那么 lowerdir 和 upperdir 目录中的内容将会合并
-   当用户修改 mergedir 中来自 upperdir 的文件时，修改直接写入 upperdir 中原来目录中，删除文件也同理
-   当用户修改 mergedir 中来自 lowerdir 的文件时，lowerdir 的文件不变。overlayFS 会先把 lowerdir 的文件拷贝一份到 upperdir，然后修改 upperdir 里的这个副本，lowerdir 里的原文件将被隐藏
-   workdir 用来存放文件中间修改过程中产生的临时文件。必须是空目录，且与 upperdir 在相同的文件系统中

####    文件读取规则
自上而下查找规则，即当容器读取文件时，OverlayFS 按以下顺序查找：
1.  upperdir（可写层）
2.  lowerdir（从顶层到底层依次查找）

合并目录：若目录在多个层中存在，其内容会被合并显示，同名文件以 upperdir 优先

####   文件写入规则
包含如下几种情况
-   修改现有文件：包含两种情况，若文件已存在于 upperdir，直接修改；若文件仅存在于 lowerdir，会触发 Copy-on-Write策略，即将文件复制到 upperdir，后续修改仅在 upperdir 中进行
-   创建新文件：直接在 upperdir 中创建
-   删除文件或目录：包含两种情况，若删除 lowerdir 中的文件，在 upperdir 创建 **.wh.<filename>** whiteout文件，标记删除；若删除 upperdir 中的文件则直接移除，不影响 lowerdir

####  overlayFS 对于docker的意义
Docker 使用 OverlayFS 作为其联合文件系统，通过分层结构和写时复制（Copy-on-Write）机制实现高效的镜像和容器管理。从分层角度来看，基于overlayFS的Docker的层次如下

-   lowerdir（只读层）：由镜像的多个只读层组成（如基础镜像、应用层等），按顺序堆叠
-   upperdir（可写层）：容器独有的可写层，所有修改均在此进行
-   merged：联合挂载后的统一视图，用户看到的是各层合并后的文件系统
-   workdir：OverlayFS 内部使用的临时目录，用于处理文件操作（如原子性重命名）

小结下：
-   镜像不可变性：所有镜像层只读，确保一致性
-   容器可写性：通过 upperdir 隔离容器修改，写时复制减少冗余操作，仅在首次修改时复制文件
-   高效存储：共享镜像层，减少冗余；多个容器共享同一组只读层，节省磁盘空间
-   隔离性：每个容器的修改独立保存在 upperdir，互不影响


##  0x01    Docker与OverlayFS

####    docker commit


##  0x0  参考
-   [Overlay 文件系统介绍](https://flyflypeng.tech/%E4%BA%91%E5%8E%9F%E7%94%9F/2023/03/29/Overlay-%E6%96%87%E4%BB%B6%E7%B3%BB%E7%BB%9F.html)
-   [docker容器技术基础之联合文件系统OverlayFS](https://zhuanlan.zhihu.com/p/392508816)
-   [理解docker [三] - git与overlayfs](https://zhuanlan.zhihu.com/p/144616121)