---
layout:     post
title:      微服务中的缓存（三）：缓存一致性实践
subtitle:   一道美团的面试题：Redis与MySQL双写一致性如何保证？
date:       2022-01-01
author:     pandaychen
header-img:
catalog: true
category:   false
tags:
    - 缓存
    - Redis
    - Mysql
    - Cache
---


##  0x00    前言
前两篇文章介绍了微服务中的缓存常用概念及一致性的问题解决，本文以go-zero项目为例，看下其针对于多级缓存的实现分析。
-   [微服务中的缓存（一）：Cache 使用与优化](https://pandaychen.github.io/2020/02/22/A-CACHE-STUDY/)
-   [微服务中的缓存（二）：微服务基础之多级服务缓存（Cache）](https://pandaychen.github.io/2020/04/10/MICROSVC-CACHE-INTRO/)

##  0x01    问题


##  0x02    


##  0x0 参考
-   [美团二面：Redis与MySQL双写一致性如何保证？](https://juejin.cn/post/6964531365643550751)
-   [go-zero缓存设计之持久层缓存](https://go-zero.dev/cn/docs/blog/cache/redis-cache/)
-   [go-zero微服务实战系列（五、缓存代码怎么写）](https://segmentfault.com/a/1190000042018127)
-   [go-zero微服务实战系列（六、缓存的一致性如何保证）](https://learnku.com/articles/69446)
-   [【原创】分布式之数据库和缓存双写一致性方案解析](https://www.cnblogs.com/rjzheng/p/9041659.html)