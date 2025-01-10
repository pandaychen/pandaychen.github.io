---
layout:     post
title:  基于golang的systemd守护进程应用分析
subtitle:
date:       2024-09-05
author:     pandaychen
header-img:
catalog: true
tags:
    - Linux
    - Systemd
---

##  0x00    前言
场景如下：
1.  服务开机启动
2.  需要时刻监测某个重要服务是否在线，如果程序报错退出则自动重启服务，以此来保持服务常在线

####    systemd简介

1、基本操作，假设服务程序路径部署在`/usr/local/bin/systemd-test`，配置存储在`/etc/systemd/system/systemd-test.service`

```BASH
[Unit]
Description=Systemd Test
After=network.target

[Service]
User=nobody
# Execute `systemctl daemon-reload` after ExecStart= is changed.
ExecStart=/usr/local/bin/systemd-test
[Install]
WantedBy=multi-user.target
```

```BASH
# 每一次修改ExecStart都需执行   
systemctl daemon-reload
# 启动
systemctl start systemd-test.service
# 查看状态
systemctl status systemd-test.service
```

##  0x01    Nginx配置



##  0x02    两个例子

1、case1:

```GO
func PANIC() {
        time.Sleep(30 * time.Second)
        panic("PANIC")
}

func main() {
        l, err := net.Listen("tcp", ":8081")
        if err != nil {
                log.Panicf("cannot listen: %s", err)
        }

        go PANIC()

        http.Serve(l, nil) 
}
```

```BASH
[root@VM-X-X-centos notify]# cat /etc/systemd/system/systemd-test.service 
[Unit]
Description=Systemd Test
After=network.target

[Service]
Type=simple
User=nobody
# Execute `systemctl daemon-reload` after ExecStart= is changed.
ExecStart=/usr/local/bin/systemd-test
Restart=on-failure
RestartSec=10
[Install]
WantedBy=multi-user.target
```

2、case2

```GO
func a() {
        for {
                _, err := http.Get("http://127.0.0.1:8081") // ❸
                if err == nil {
                        daemon.SdNotify(false, daemon.SdNotifyWatchdog)
                }
                time.Sleep(3 * time.Second)
        }
}

func b() {
        time.Sleep(30 * time.Second)
        panic("panic")
}

func main() {
        l, err := net.Listen("tcp", ":8081")
        if err != nil {
                log.Panicf("cannot listen: %s", err)
        }
        daemon.SdNotify(false, daemon.SdNotifyReady) 

        go a()

        go b()

        http.Serve(l, nil) 
}
```

```bash
[root@VM-X-X-centos notify]# cat /etc/systemd/system/systemd-test.service 
[Unit]
Description=Systemd Test
After=network.target

[Service]
Type=notify
User=nobody
# Execute `systemctl daemon-reload` after ExecStart= is changed.
ExecStart=/usr/local/bin/systemd-test
Restart=on-failure
RestartSec=10
[Install]
WantedBy=multi-user.target
```

##  0x0 参考
-   [Go bindings to systemd socket activation, journal, D-Bus, and unit files](https://github.com/coreos/go-systemd)
-   [Golang应用结合systemd注册服务为守护进程](https://blog.csdn.net/C_0010/article/details/131484981)
-   [利用 systemd 部署 golang 项目](https://learnku.com/articles/34025)
-   [Integration of a Go service with systemd: readiness & liveness](https://vincent.bernat.ch/en/blog/2017-systemd-golang)