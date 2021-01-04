---
layout:     post
title:      Confd 使用 && 源码分析
subtitle:   Confd 源码分析：强大的动态配置更新（基于 etcdv3/redis 存储）
date:       2021-01-01
author:     pandaychen
header-img: img/super-mario.jpg
catalog: true
category:   false
tags:
    - Confd
    - Golang
---

##  0x00    前言
现网有类似下面的 Nginx 配置，如何对 `upstream` 指向的后端服务节点做动态的上下线切换（即时剔除无效节点）？一个可行的解决方案就是使用 Confd+Etcd。
```javascript
upstream backend_cluster {
    server 172.19.161.1:9081;
    server 172.19.161.2:9081;
    server 172.19.161.3:9081;
}

server {
        listen       8080;
        #..
        location ^~ /cgi/ {
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header Cookie $http_cookie;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host $host;
        proxy_pass http://backend_cluster/;
        proxy_pass_request_headers      on;
    }
}
```

最近准备给 Nginx 代理的配置文件提供动态更新 reload 的功能，采用 Etcd+Confd 实现，本文简单分析下 [Confd](https://github.com/kelseyhightower/confd) 的实现。
Confd 是一个轻量级的配置管理工具，可以通过查询后端存储系统来实现第三方应用的（动态）配置管理，如 Nginx、HAproxy、Docker 配置等。Confd 能够查询和监听后端系统的数据变更，结合配置模版引擎动态更新本地配置文件，保持和后端系统的数据一致，并且能够执行命令或者脚本实现系统的 reload 或者 restart 等操作。

##  0x01    Confd 的使用
1、配置文件 `confd.toml`，主要记录了使用的存储后端、`confdir` 等，参数 `watch` 表示实时监听 etcdV3 的变化，如有变化则更新 Confd 管理的配置（推荐使用）<br>
```toml
backend = "etcdv3"
confdir = "/etc/confd"
log-level = "debug"
interval = 5
nodes = [
  "http://127.0.0.1:2379",
]
scheme = "http"
watch = true
```

####    confdir 的目录配置
包含配置与模板两个目录：
-   `./conf.d/`：Confd 的配置文件，包含配置的生成逻辑
-   `./templates`：配置模板 Template，即基于不同应用组件的配置，遵循 Golang text templates 的模板文件，详见 [文档](https://github.com/kelseyhightower/confd/blob/master/docs/templates.md)

例如：

####    同步生成 Nginx 配置


##  0x02    Confd 分析
Confd 的整体应用架构如下：
![img](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/etcd/confd-architecture.png)

##  参考
-   [Quick Start Guide](https://github.com/kelseyhightower/confd/blob/master/docs/quick-start-guide.md)
