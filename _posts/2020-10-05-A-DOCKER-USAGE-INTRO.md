---
layout:     post
title:      Docker 日常实践汇总
subtitle:   工作中的经验总结
date:       2020-10-05
author:     pandaychen
header-img:
catalog: true
category:   false
tags:
    - Docker
---

##  0x00    前言
本文介绍下日常工作的一些基础 Docker 使用

##  0x01    docker 镜像制作和镜像打包

####    问题
作为一个入门级用户，很有可能写出下面的 Dockerfile：

```BASH
From centos:7

LABEL maintainer="panda<ringbuffer@126.com>"

ADD requirements.txt /
ADD xxxx.sh    /
ADD xxxx-1.2.3.4.tar.gz /    ## 自动解压
ADD xxxx-master.zip /


WORKDIR /

RUN     export http_proxy=http://xxx.xx.com:8080 && \
        export https_proxy=http://xxx.xx.com:8080 && \
        yum install -y yum-utils        && \
        yum-config-manager --enable epel       && \
        yum -y install unzip zip && \
        unzip xxxx-master.zip && \
        mkdir -p /data/ &&      \
        mkdir -p /data/log/&& \
        yum install -y epel-release && \
        yum install -y yum-utils        && \
        yum -y install python && \
        yum -y install python-pip && \
        pip install --upgrade pip       && \
        yum -y install python-devel mysql-devel && \
        yum install gcc -y && \
        pip install --upgrade pip && \
        pip install -r requirements.txt && \
        cd xxxx-1.2.3.4 && \
        python setup.py install && \
        unzip /xxxx-master.zip && \
        cd /xxxx/ && \
        python setup.py install && \
        pip install cryptography &&\
        export http_proxy= && \
        export https_proxy= && \
        unset http_proxy && \
        unset https_proxy

CMD ["/bin/sh", "-c","cd /xxxx/bin/ && /usr/bin/python xxxxx.py"]
```

上面这个 Dockerfile 的问题在于，流程过于冗长，每次调用该 Dockerfile 生成镜像会耗费较长时间，那么如何优化呢？不考虑更换 `From centos:7` 的情况下，可以通过镜像制作的方式来解决，步骤如下：

####    制作 && 打包

```BASH
docker pull centos:7    #拉取一个基础镜像
docker run -it --name=bfcentos centos:7   #创建一个交互式容器

docker cp  xxx.tar  bfcentos:/data/     #上传一些公共包到容器的 / data 目录

#在容器中安装软件

docker commit bfcentos bfcentos-python27  #将交互式的容器提交为一个新的镜像
```

通过 `docker commit` 将刚才部署好公共软件 / 包的容器提交为一个新的镜像，然后打上 tag 推送到公司的镜像仓库去：

```text
[root@VM_120_245_centos /data/rootback/docker-python]# docker commit bfcentos bfcentos-python27
sha256:xxxxxxx

[root@VM_120_245_centos /data/rootback/docker-python]# docker tag bfcentos-python27:latest mirrors.xxxx.com/samp/bfcentos-python27:latest
[root@VM_120_245_centos /data/rootback/docker-python]# docker push  mirrors.xxxx.com/samp/bfcentos-python27:latest
The push refers to repository [mirrors.xxxx.com/samp/bfcentos-python27]
f77c094d51c9: Pushing [=========>                                         ]  73.25MB/404.9MB
174f56854903: Pushing [=================>                                 ]   70.4MB/203.9MB
```

这样，下次可以直接使用 `From mirrors.xxxx.com/samp/bfcentos-python27:latest` 构建：

```BASH
From mirrors.xxxx.com/samp/bfcentos-python27:latest

LABEL maintainer="panda<ringbuffer@126.com>"

ADD xxxx-master.zip /

WORKDIR /

RUN     unzip /xxxx-master.zip
CMD ["/bin/sh", "-c","cd /xxxx/bin/ && /usr/bin/python xxxx.py"]
```

####    镜像 && 容器打包
```bash
docker save -o /root/xxxx.tar  <name>        #镜像打包
docker load -i /root/xxxx.tar               #导入镜像
docker export -o /root/xxxx.tar  <name>   #容器打包
docker import xxxx.tar <name>:latest  #导入容器
```