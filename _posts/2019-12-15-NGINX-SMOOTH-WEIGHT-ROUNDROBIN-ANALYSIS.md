---
layout:     post
title:      Nginx负载均衡及算法分析
subtitle:   
date:       2019-12-15
author:     pandaychen
header-img: img/encryption.jpg
catalog: true
tags:
    - Loadbalance
    - Nginx
    - 负载均衡
    - 平滑负载均衡算法
---

##  0x00    引入问题
提一个问题：Nginx的`upstream`机制支持自动剔除故障的后端吗？这个问题我们后面解答

##	0x01	Nginx中的负载均衡介绍

2019下半年，笔者在私有化Sass项目，使用[`Nginx C Module`](https://github.com/nginx-modules/nginx-c-function)`+`[企业微信`API`](https://work.weixin.qq.com/api/doc/90001/90142/90594)实现了一个`OAuth`认证网关（提供`Web/Api`调用鉴权），过程中对Nginx的模块开发、负载均衡`upstream`、`proxy-pass`等一些特性，有了一些深入的了解。这篇博客就来聊聊Nginx中的负载均衡机制

### 介绍
Nginx提供了[`upstream`](http://tengine.taobao.org/book/chapter_05.html)机制，来实现访问后端时的负载均衡。模块的具体使用说明（翻译版本）在此: [`ngx_http_upstream_module`](https://tengine.taobao.org/nginx_docs/cn/docs/http/ngx_http_upstream_module.html)

### 负载均衡模式
* weight roundrobin（内置）- 带权重的轮询
	-	默认方式，每个请求按时间顺序逐一分配到不同的后端服务器，如果超过了最大失败次数后（`max_fails`,默认1），在失效时间内(`fail_timeout`，默认10秒)，该节点失效权重变为`0`，超过失效时间后，则恢复正常，或者全部节点都为down后，那么将所有节点都恢复为有效继续探测。Nginx实现了一个非常优秀的[带权重的负载均衡算法](https://github.com/phusion/nginx/commit/27e94984486058d73157038f7950a0a36ecc6e35)，此算法具备极好的平滑性
* ip_hash（内置）
	-	每个请求按访问ip的hash结果分配，这样每个访客固定访问一个后端服务器，可以解决session的问题，但是ip_hash会造成负载不均，有的服务请求接受多，有的服务请求接受少，所以不建议采用ip_hash模式，session共享问题可用后端服务的session共享代替nginx的ip_hash
* fair（扩展）
	-	按后端服务器的响应时间来分配请求，响应时间短的优先分配
* url_hash（扩展）
	-	和ip_hash算法类似，是对每个请求按`url`的hash结果分配，使每个`url`定向到一个同一个后端Server

### 负载均衡指令及含义
定义一组用于实现nginx负载均衡的服务器，它们可以侦听在不同的端口。常用的配置方式如下：

```bash
#在http节点下，加入upstream节点
upstream backend { 
    server 1.2.3.4:7080; 
    server 1.2.3.5:7080; 
    server 1.2.3.6:7080; 
}
#将server节点下的location节点中的proxy_pass配置为：http:// + upstream名称，即http://linuxidc
location / { 
    root  html; 
    index  index.html index.htm; 
    proxy_pass http://backend; 
}
```

``` bash
upstream backend {
    server backend1.example.com       weight=5;
    server backend2.example.com:8080;
    server unix:/tmp/backend3;
	server	1.2.3.4:8081;

    server backup1.example.com:8080   backup;
    server backup2.example.com:8080   backup;
}

server {
    location / {
        proxy_pass http://backend;		#We USE proxy_pass
    }
}
```

##### server 地址 [参数]；
可定义如下参数：
* `weight`=number：权重
* `max_fails`=number：设置在`fail_timeout`参数设置的时间内最大失败次数，如果在这个时间内，所有针对该服务器的请求都失败了，那么认为该服务器会被认为是停机了，停机时间是`fail_timeout`设置的时间。默认情况下，不成功连接数被设置为1。被设置为零则表示不进行链接数统计。
* `fail_time`=time：如上
* `backup`: 标记该服务器为备用服务器。当主服务器停止时，请求会被发送到此
* `down`: 标记服务器永久停机了；与指令ip_hash一起使用

###	与Kubernetes的结合
待补充

### 典型的负载均衡策略配置

##### Round Robin + Weighted Round Robin
-   普通轮询：每个请求按时间顺序（全局Index）逐一分配到不同的后端服务器，如果后端服务器down掉，能自动剔除故障的后端吗？
-   加权轮询：机器按照权重权限, 首先将请求都分配给权重最高的机器, 类似一种「深度优先」, 如果该机器成功建立连接 将会把权重减一重排。如果后端某台服务器宕机，故障系统被自动剔除。当所有后端机器都down掉时，nginx会立即将所有机器的标志位清成初始状态，以避免造成所有的机器都处在timeout的状态，从而导致整个前端被夯住

```bash
upstream backendpoint1 {
  server 1.1.1.1:8080;
  server 2.2.2.2:8080;
}

upstream backendpoint2 {
  server 1.1.1.1:8080 weight=10;
  server 2.2.2.2:8080 weight=10;
}
```
##### Ip Hash
-   基于客户端ip的负载均衡算法, IPV4的前3个八进制位和所有的IPV6地址被用作一个hashkey

```bash
upstream backendpoint3 {
  ip_hash;
  server 1.1.1.1:8080;
  server 2.2.2.2:8081;
}
```

hash值既与ip有关又与后端机器的数量有关, hash 算法可以产生1045个互异的value.

来自同一个IP的访客固定访问一个后端服务器，有效解决了动态网页存在的session共享问题。当然如果这个节点不可用了，会发到下个节点，而此时没有session同步的话就注销掉了

在实际的网络环境中，有大量的高校出口路由器ip、企业出口路由器ip等网络节点，这些节点带来的流量往往是普通用户的成百上千倍

如果nginx负载均衡器组里面的一个服务器要临时移除，它应该用参数down标记，来防止之前的客户端IP还往这个服务器上发请求

### fair

第三方负载均衡策略

```
upstream backserver {
  server server1;
  server server2;
  fair;
}
```

扩展策略, 默认不被编译进nginx内核, 其原理是根据后端服务器的响应时间判断负载情况，从中选出负载最轻的机器进行分流.

### url_hash

第三方负载均衡策略

按访问url的hash结果来分配请求，使每个url定向到同一个后端服务器，后端服务器为缓存时比较有效.

```
upstream backserver {
  server squid1:3128;
  server squid2:3128;
  hash $request_uri;
  hash_method crc32;
}
```


### session_sticky

依赖第三方模块 `nginx-sticky-module`

`sticky [name=route] [domain=.foo.bar] [path=/] [expires=1h] [hash=index|md5|sha1] [no_fallback];`

通过cookie黏贴的方式将来自同一个客户端（浏览器）的请求发送到同一个后端服务器上处理

这个模块并不合适不支持 Cookie 或手动禁用了cookie的浏览器，此时默认sticky就会切换成RR。它不能与ip_hash同时使用

`ngx_http_upstream_session_sticky_module` 也是类似的功能

## Nginx后端服务器的容错
先给一个结论，nginx的upstream容错（健康检查）机制是有限的。nginx自带是没有针对负载均衡后端节点的健康检查的，但是可以通过默认自带的`ngx_http_proxy_module`模块和 `ngx_http_upstream_module`模块中的相关指令来完成当后端节点出现故障时，自动切换到下一个节点来提供访问

* proxy_next_upstream

[`nginx_upstream_check_module`](https://github.com/yaoweibin/nginx_upstream_check_module) 是专门提供负载均衡器内节点的健康检查的第三方模块

通过它可以用来检测后端 realserver 的健康状态。如果后端 realserver 不可用，则后面的请求就不会转发到该节点上

####	upstream容错机制介绍

1.	Nginx 判断节点失效状态
Nginx默认判断失败节点状态以connect refuse和time out状态为准，不以HTTP错误状态进行判断失败，因为HTTP只要能返回状态说明该节点还可以正常连接，所以nginx判断其还是存活状态；除非添加了proxy_next_upstream指令设置对404、502、503、504、500和time out等错误进行转到备机处理，在next_upstream过程中，会对fails进行累加，如果备用机处理还是错误则直接返回错误信息（但404不进行记录到错误数，如果不配置错误状态也不对其进行错误状态记录），综述，nginx记录错误数量只记录timeout 、connect refuse、502、500、503、504这6种状态，timeout和connect refuse是永远被记录错误状态，而502、500、503、504只有在配置proxy_next_upstream（下面介绍）后nginx才会记录这4种HTTP错误到fails中，当fails大于等于max_fails时，则该节点失效

2.	Nginx 处理节点失效和恢复的触发条件
nginx可以通过设置max_fails（最大尝试失败次数）和fail_timeout（失效时间，在到达最大尝试失败次数后，在fail_timeout的时间范围内节点被置为失效，除非所有节点都失效，否则该时间内，节点不进行恢复）对节点失败的尝试次数和失效时间进行设置，当超过最大尝试次数或失效时间未超过配置失效时间，则nginx会对节点状会置为失效状态，nginx不对该后端进行连接，直到超过失效时间或者所有节点都失效后，该节点重新置为有效，重新探测；

3.  所有节点失效后nginx将重新恢复所有节点进行探测
如果探测所有节点均失效，备机也为失效时，那么nginx会对所有节点恢复为有效，重新尝试探测有效节点，如果探测到有效节点则返回正确节点内容，如果还是全部错误，那么继续探测下去，当没有正确信息时，节点失效时默认返回状态为502，但是下次访问节点时会继续探测正确节点，直到找到正确的为止。

####    proxy_next_upstream机制
Nginx通过[`proxy_next_upstream`](http://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_next_upstream)指令实现容灾处理
ngx_http_proxy_module 模块中包括proxy_next_upstream指令
语法: proxy_next_upstream error | timeout | invalid_header | http_500 | http_502 | http_503 | http_504 |http_404 | off ...; 
默认值: proxy_next_upstream error timeout; 
上下文: http, server, location

其中：
error   表示和后端服务器建立连接时，或者向后端服务器发送请求时，或者从后端服务器接收响应头时，出现错误。
timeout   表示和后端服务器建立连接时，或者向后端服务器发送请求时，或者从后端服务器接收响应头时，出现超时。
invalid_header   表示后端服务器返回空响应或者非法响应头 
http_500   表示后端服务器返回的响应状态码为500 
http_502   表示后端服务器返回的响应状态码为502 
http_503   表示后端服务器返回的响应状态码为503 
http_504   表示后端服务器返回的响应状态码为504 
http_404   表示后端服务器返回的响应状态码为404 
off   表示停止将请求发送给下一台后端服务器

1）proxy_next_upstream http_500 | http_502 | http_503 | http_504 |http_404;
当其中一台返回错误码404,500...等错误时，可以分配到下一台服务器程序继续处理，提高平台访问成功率，多可运用于前台程序负载，设置
 
2、proxy_next_upstream off
因为proxy_next_upstream 默认值: proxy_next_upstream error timeout;
 
场景:
当访问A时，A返回error timeout时，访问会继续分配到下一台服务器处理，就等于一个请求分发到多台服务器，就可能出现多次处理的情况，
如果涉及到充值，就有可能充值多次的情况，这种情况下就要把proxy_next_upstream关掉即可
proxy_next_upstream off
 
案例分析（nginx proxy_next_upstream导致的一个重复提交错误）：
一个请求被重复提交，原因是nginx代理后面挂着2个服务器，请求超时的时候（其实已经处理了），结果nigix发现超时，有把请求转给另外台服务器又做了次处理。
 
解决办法：
proxy_next_upstream:off

#####  proxy_next_upstream的运用场景
我们看如下的配置，当时的设想是，如果这个服务的访问，出现了500或者超时的情况，会自动重试到下一个服务器去，采用的是nginx的proxy_next_upstream, 配置大概如下：
按照设想，如果192.168.0.1这台服务器返回了超时或者500，那么访问 /example/ 时应该会自动重试到下一台服务器去，即到192.168.0.2去。

``` bash
upstream example_upstream{
  server 192.168.0.1 max_fails=1 fail_timeout=30s;
  server 192.168.0.2 max_fails=1 fail_timeout=30s backup;
}
location /example/ {
   proxy_pass http://example_upstream/;
   proxy_set_header Host: test.example.com;
   proxy_set_head X-real-ip $remote_addr;
   proxy_next_upstream error timeout http_500 non_idemponent;
}
```

##  0x02    Nginx对Round-Robin算法的优化

Smooth Weighted Round-Robin (SWRR) 是 nginx 默认的加权负载均衡算法，它的重要特点是平滑，避免低权重的节点长时间处于空闲状态，因此被称为平滑加权轮询。
该算法来自 nginx 的一次 commit：Upstream: smooth weighted round-robin balancing

nginx使用的平滑权重轮询算法介绍以及原理分析。
<!-- more -->
# 轮询调度
轮询调度非常简单，就是每次选择下一个节点进行调度。比如`{a, b, c}`三个节点，第一次选择a，
第二次选择b，第三次选择c，接下来又从头开始。

这样的算法有一个问题，在负载均衡中，每台机器的性能是不一样的，对于16核的机器跟4核的机器，
使用一样的调度次数，这样对于16核的机器的负载就会很低。这时，就引出了基于权重的轮询算法。

基于权重的轮询调度是在基本的轮询调度上，给每个节点加上权重，这样对于权重大的节点，
其被调度的次数会更多。比如a, b, c三台机器的负载能力分别是4:2:1，则可以给它们分配的权限为4, 2, 1。
这样轮询完一次后，a被调用4次，b被调用2次，c被调用1次。

对于普通的基于权重的轮询算法，可能会产生以下的调度顺序`{a, a, a, a, b, b, c}`。  

这样的调度顺序其实并不友好，它会一下子把大压力压到同一台机器上，这样会产生一个机器一下子很忙的情况。
于是乎，就有了平滑的基于权重的轮询算法。

所谓平滑就是调度不会集中压在同一台权重比较高的机器上。这样对所有机器都更加公平。
比如，对于`{a:5, b:1, c:1}`，产生`{a, a, b, a, c, a, a}`的调度序列就比`{c, b, a, a, a, a, a}`
更加平滑。

##	0x04 nginx平滑的基于权重轮询算法
nginx平滑的基于权重轮询算法其实很简单。[算法原文](https://github.com/phusion/nginx/commit/27e94984486058d73157038f7950a0a36ecc6e35)
描述为：
> Algorithm is as follows: on each peer selection we increase current_weight
> of each eligible peer by its weight, select peer with greatest current_weight
> and reduce its current_weight by total number of weight points distributed
> among peers.

算法执行2步，选择出1个当前节点。  
1. 每个节点，用它们的当前值加上它们自己的权重。
2. 选择当前值最大的节点为选中节点，并把它的当前值减去所有节点的权重总和。

例如`{a:5, b:1, c:1}`三个节点。一开始我们初始化三个节点的当前值为`{0, 0, 0}`。
选择过程如下表：  

| 轮数 | 选择前的当前权重 | 选择节点 | 选择后的当前权重 |
|------|------------------|----------|------------------|
| 1    | {5, 1, 1}        | a        | {-2, 1, 1}       |
| 2    | {3, 2, 2}        | a        | {-4, 2, 2}       |
| 3    | {1, 3, 3}        | b        | {1, -4, 3}       |
| 4    | {6, -3, 4}       | a        | {-1, -3, 4}      |
| 5    | {4, -2, 5}       | c        | {4, -2, -2}      |
| 6    | {9, -1, -1}      | a        | {2, -1, -1}      |
| 7    | {7, 0, 0}        | a        | {0, 0, 0}        |

我们可以发现，a, b, c选择的次数符合5:1:1，而且权重大的不会被连接选择。7轮选择后，
当前值又回到{0, 0, 0}，以上操作可以一直循环，一样符合平滑和基于权重。

##  0x05  Nginx对gRPC负载均衡的支持
Nginx从1.13.10版本[Introducing gRPC Support with NGINX 1.13.10](http://manguijie.top/2018/09/grpc-name-resolve-loadbalance)宣布支持gRPC负载均衡。
nginx配置中增加`grpc_pass`，使用方式与`proxy_pass`基本一致，如下：
```bash
worker_processes  1;
events {
    worker_connections  1024;
}

http {
    include       mime.types;
    default_type  application/octet-stream;
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    error_log   logs/error.log debug;
    access_log  logs/access.log  main;

    sendfile        on;
    keepalive_timeout  65;

    upstream grpc_servers {
        #least_conn;
        #keepalive 1000;
        server 127.0.0.1:10006;
    }
    server {
        listen       8080 http2;
        server_name  localhost;
        location / {
            grpc_pass grpc://grpc_servers;
            error_page 502 = /error502grpc;
        }

        location = /error502grpc {
            internal;
            default_type application/grpc;
            add_header grpc-status 14;
            add_header grpc-message "unavailable";
            return 204;
        }
    }
}
```

##  0x05    回顾&&总结
&emsp;&emsp;下面来回答下本篇一开始提出的问题：

此外，Nginx的收费版本[`Nginx-Plus`](https://www.nginx.com/products/nginx/)提供了自动探测的能力


##	0x07	参考
-	[nginx平滑的基于权重轮询算法分析](https://tenfy.cn/2018/11/12/smooth-weighted-round-robin/)
-   [加权轮询策略剖析](http://blog.csdn.net/xiajun07061225/article/details/9318871)
-   [解析 Nginx 负载均衡策略](http://www.cnblogs.com/wpjamer/articles/6443332.html)
-   [Nginx做负载均衡器以及Proxy缓存配置](https://mp.weixin.qq.com/s?__biz=MzA3MzYwNjQ3NA==&mid=401736802&idx=1&sn=9508d9a05b725c66a05e05d8dad6dec1#rd)


转载请注明出处，本文采用 [CC4.0](http://creativecommons.org/licenses/by-nc-nd/4.0/) 协议授权