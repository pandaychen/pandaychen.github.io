---
layout:     post
title:      Cache 设计、使用与优化
subtitle:   如何在设计高效的缓存使用策略
date:       2020-02-22
author:     pandaychen
header-img:
catalog: true
category:   false
tags:
    - Cache
---

##  0x00    前言

##  Cache 分类
*   远程 Cache

*   本地 Cache

##  一些优秀的开源 Local Cache 项目
-   [go-cache](https://github.com/patrickmn/go-cache)
> An in-memory key:value store/cache (similar to Memcached) library for Go, suitable for single-machine applications. https://patrickmn.com/projects/go-cache/

-   [bigcache](https://github.com/allegro/bigcache)
> 非常精妙的 cache 实现，参见 https://github.com/allegro/bigcache

-   [ccache](https://github.com/karlseguin/ccache)
> A golang LRU Cache for high concurrency

##  0x  参考
-   [设计实现高性能本地内存缓存](https://blog.joway.io/posts/modern-memory-cache/)
-   [如何设计并实现一个线程安全的 Map ？](https://halfrost.com/go_map_chapter_two/)
-   [Writing a very fast cache service with millions of entries in Go](https://allegro.tech/2016/03/writing-fast-cache-service-in-go.html)