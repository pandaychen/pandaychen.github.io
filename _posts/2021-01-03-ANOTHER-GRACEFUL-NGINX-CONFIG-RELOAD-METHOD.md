---
layout:     post
title:      Nginx 容器动态流量管理方案：Upsync
subtitle:   一种更为优雅的 Nginx 代理动态切换方案
date:       2021-01-03
author:     pandaychen
header-img: img/super-mario.jpg
catalog: true
category:   false
tags:
    - Nginx
    - Etcd
    - Consul
---

##  0x00    前言
前一篇文章，介绍过 [Nginx 的负载均衡算法](https://pandaychen.github.io/2019/12/15/NGINX-SMOOTH-WEIGHT-ROUNDROBIN-ANALYSIS/)。<br>
`upstream` 机制使得 Nginx 通常用于反向代理服务器，Nginx 接收来自下游客户端的 Http 请求，并处理该请求，同时根据该请求向上游服务器发送 Tcp 请求报文，上游服务器会根据该请求返回相应地响应报文，Nginx 根据上游服务器的响应报文，决定是否向下游客户端转发响应报文。另外 `upstream` 机制提供了负载均衡的功能，可以将请求负载均衡到集群服务器的某个服务器上面。

同时，上一篇文章介绍了采用 Confd+Nginx 的 [方案](https://pandaychen.github.io/2021/01/01/A-GOLANG-CONFD-ANALYSIS/)，不过，在高并发的场景下，使用 `nginx -s reload` 方式可能会带来性能上的损耗，不够优雅。本篇介绍下 Weibo 开源的 Nginx Module：Upsync，基于 Nginx 容器动态流量管理方案。该方案结合了 Nginx 的健康检查模块，以及动态 `reload` 机制，可以近乎无损的服务的升级上线与扩容。

Upsync 是 Nginx 和 Etcd、Consul 等服务发现组件非常好的结合实践。

##  0x01    架构 && 流程
-   [nginx-upsync-module](https://github.com/weibocom/nginx-upsync-module)：支持 Http 的配置
-   [nginx-stream-upsync-module](https://github.com/xiaokai-wang/nginx-stream-upsync-module)：支持 Stream（四层）的代理配置

![nginx-upsync - 原理图]()

##  0x02    UpSync 配置及使用

####    配置指令说明
| 指令 | 位置 | 功能 | 语法示例 | 默认值 |
|------|------------|----------|----------|----------|
| upsync  |  upstream     | 从 Consul/Etcd 获取 `upstream` 服务列表 | upsync $consul.api.com:$port/v1/kv/upstreams/$upstream_name [upsync_type=consul] [upsync_interval=seconds/minutes] [upsync_timeout=seconds/minutes] [strong_dependency=off/on];|upsync_interval=5s upsync_timeout=6m strong_dependency=off|


####    配置示例
先来看一个 Upsync 配置示例：
```JavaScript
worker_processes  4;

load_module modules/ngx_http_upsync_module.so;

events {
    worker_connections  1024;
}

http {
    include       mime.types;
    default_type  application/octet-stream;

    log_format  main  '$remote_addr - $remote_user [$time_local]"$request" '
                      '$status $body_bytes_sent"$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  logs/access.log  main;

    upstream test {
        upsync 127.0.0.1:2379/v2/keys/upstreams/pool upsync_timeout=6m upsync_interval=500ms upsync_type=etcd strong_dependency=off;
        upsync_dump_path /etc/nginx/server_list.conf;
        include /etc/nginx/server_list.conf;
    }

    upstream bar {
        server 127.0.0.1:8090 weight=1 fail_timeout=10 max_fails=3;
    }

    server {
        listen 80;

        location = /index.html {
            proxy_pass http://test;
        }

        location = /bar {
            proxy_pass http://bar;
        }

        location = /upstream_show {
            upstream_show;
        }
    }
}
```

初始化 server 列表如下：
```bash
cat /etc/nginx/server_list.conf
server 127.0.0.1:8080 weight=1 max_fails=2 fail_timeout=10s;
server 127.0.0.1:8081 weight=1 max_fails=2 fail_timeout=10s;
```

通过 ETCD V2 版本接口进行数据操作：
```JavaScript
#增加 upstream 节点，默认属性为：weight=1 max_fails=2 fail_timeout=10 down=0 backup=0;
curl -X PUT http://127.0.0.1:2379/v2/keys/upstreams/pool/127.0.0.1:8082
#删除数据
curl -X DELETE http://127.0.0.1:2379/v2/keys/upstreams/pool/127.0.0.1:8080
#获取数据
curl http://127.0.0.1:2379/v2/keys/upstreams/pool
#增加节点，或者修改属性
curl -X PUT -d value='{"weight":1,"max_fails":2,"fail_timeout":10}' http://127.0.0.1:2379/v2/keys/upstreams/pool/127.0.0.1:8081
#标记节点下线
curl -X PUT -d value='{"weight":2,"max_fails":2,"fail_timeout":10,"down":1}' http://127.0.0.1:2379/v2/keys/upstreams/pool/127.0.0.1:8082
#查看当前 upstream
http://127.0.0.1/upstream_show
Upstream name: test; Backend server count: 2
        server 127.0.0.1:8081 weight=1 max_fails=2 fail_timeout=10s;
        server 127.0.0.1:8082 weight=1 max_fails=2 fail_timeout=10s;

Upstream name: bar; Backend server count: 1
        server 127.0.0.1:8090 weight=1 max_fails=3 fail_timeout=10s;
```

##  0x03    核心代码分析
整个模块的核心逻辑在 [ngx_http_upsync_module.c](https://github.com/weibocom/nginx-upsync-module/blob/master/src/ngx_http_upsync_module.c)：

`ngx_http_client_send` 函数，调用 GET 请求访问 Consul 或者 Etcd 集群并获取结果：
```c
static ngx_int_t
ngx_http_client_send(ngx_http_conf_client *client,
    ngx_http_upsync_server_t *upsync_server)
{
    size_t       size = 0;
    ngx_int_t    tmp_send = 0;
    ngx_uint_t   send_num = 0;

    ngx_upsync_conf_t           *upsync_type_conf;
    ngx_http_upsync_srv_conf_t  *upscf;

    upscf = upsync_server->upscf;
    upsync_type_conf = upscf->upsync_type_conf;

    u_char request[ngx_pagesize];
    ngx_memzero(request, ngx_pagesize);

    if (upsync_type_conf->upsync_type == NGX_HTTP_UPSYNC_CONSUL
        || upsync_type_conf->upsync_type == NGX_HTTP_UPSYNC_CONSUL_SERVICES
        || upsync_type_conf->upsync_type == NGX_HTTP_UPSYNC_CONSUL_HEALTH)
    {
        ngx_sprintf(request, "GET %V?recurse&index=%uL HTTP/1.0\r\nHost: %V\r\n"
                    "Accept: */*\r\n\r\n",
                    &upscf->upsync_send, upsync_server->index,
                    &upscf->conf_server.name);
    }

    if (upsync_type_conf->upsync_type == NGX_HTTP_UPSYNC_ETCD) {
        ngx_sprintf(request, "GET %V? HTTP/1.0\r\nHost: %V\r\n"
                    "Accept: */*\r\n\r\n",
                    &upscf->upsync_send, &upscf->conf_server.name);
    }

    //...
}
```

##  参考
-   [Nginx 开发从入门到精通：upstream 模块](http://tengine.taobao.org/book/chapter_5.html)
-   [Nginx 中 upstream 机制的实现](https://www.kancloud.cn/digest/understandingnginx/202606)
-   [Upsync：微博开源基于 Nginx 容器动态流量管理方案](https://mp.weixin.qq.com/s?__biz=MzAwMDU1MTE1OQ==&mid=404151075&idx=1&sn=5f3b8c007981a2d048766f808f8c8c98&scene=2&srcid=0223XScbJrOv7noogVX6T60Q&from=timeline&isappinstalled=0#wechat_redirect)