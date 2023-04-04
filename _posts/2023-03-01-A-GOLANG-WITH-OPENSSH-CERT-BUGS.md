---
layout:     post
title:      OpenSSH Certificate 与 Golang 的兼容性问题
subtitle:
date:       2023-03-01
author:     pandaychen
header-img: img/golang-horse-fly.png
catalog: true
category:   false
tags:
    - OpenSSH
    - Golang
---


##  0x00    前言
前文 [OpenSSH Certificate 证书最佳实践](https://pandaychen.github.io/2020/04/09/OPENSSH-CERT-BEST-PRACTISE/) 介绍了证书的实践，不过近期笔者在工作中遇到了非常麻烦的兼容性问题，这里摘录几个：

-   [x/crypto/ssh: "ssh-rsa-cert-v01@openssh.com" does not work for sshd OpenSSH 7.2-7.7 #58371](https://github.com/golang/go/issues/58371)
-   [x/crypto/ssh: cannot sign certificate with different algorithm #36261](https://github.com/golang/go/issues/36261)
-   [x/crypto/ssh: rsa-sha2-256/rsa-sha2-512 tracking issue](https://github.com/golang/go/issues/49952#issuecomment-1314152075)
-   [OpenSSH server compatibility issues #17197](https://github.com/gravitational/teleport/issues/17197)
-   [x/crypto/ssh: support RSA SHA-2 host key signatures #37278](https://github.com/golang/go/issues/37278)

以上的问题核心点是兼容性问题，主要涉及到三方的兼容性问题：
1.  golang（`golang.org/x/crypto/ssh`）版本（也有可能是二进制版本），主要影响两块：
    -   CA 服务端：证书签发（不同 go 版本编译的生成证书的格式可能不一样，如 `1.11` 和 `1.17` 编译出的 CA 服务生成的 `RSA-2048` 证书，`1.11` 的编译方式为 `GOPATH`，`1.17` 的编译方式为 `GOMOD`）
    -   SSH 客户端：证书解析及登录（主要是解析证书，`1.11` 版本的客户端不一定能解析出 `1.17` 版本生成的证书）
2.  Openssh：sshd 版本（服务器），对算法、证书类型有兼容性要求
3.  Openssh：ssh 版本（客户端），包括 `ssh-keygen` 工具的兼容性问题

##      0x01    兼容性测试
现网使用的 go 版本主要是：`go1.11` 以及 `go.17`，主要选择两种用户证书类型：`RSA-2048`、`ED25519`，CA 根证书的类型为 `RSA-2048`，sshd 版本选择三种：
-   `OpenSSH_6.0p1, OpenSSL 1.0.0-fips 29 Mar 2010`
-   `OpenSSH_7.4p1, OpenSSL 1.0.2k-fips  26 Jan 2017`
-   `OpenSSH_8.8p1, OpenSSL 1.0.2k-fips  26 Jan 2017`



`RSA-2048` 用户证书（1.11 版本服务端签发）：

```text
ssh-rsa-cert-v01@openssh.com AAAAHHNzaC1yc2EtY2VydC12MDFAb3BlbnNzaC5jb20AAAAgZMQk5qch3lX21XXtjrx8aBuBPlM8qTG17S6jJmMOVogAAAADAQABAAABAQCYHr8nSSO3iJzyUYv2ZAGx3m+jB4dv6EuRWP5MMVaSGpnQxaiWq9AJC6iQ61o44BMCvtidlPfk3crgr9MHTifWkaMQyEwpa3vmkXKBIMTYEVJOp3bAWVhl9ialdfzGQZKqE21LqCMQ5WmBOZMcpk3ntctpm4tb7ua7VK5E+4oOlA4/vZkZX5u2kzLJaAKblJNR6GEAMU7npraz0whZly27XS8DQEKMwWsW2IlVREyehKT83ji5t58a/p9uHISq1aM1siUmvvW6AVJ50x35KO8/HF9Lyo5HUh0scqmXJVL8vmMl6kHTcMVze6WInyPtEmbPQOq8tM+NGYR5E4VUYiklAAAAAAAAAAAAAAABAAAALzEwMDYjMTUwLjEwOS4xMTguMTYjMzYwMDEjcGFuZGF5Y2hlbiNyb290I25vdXNlAAAACAAAAARyb290AAAAAGOrwY8AAAAAZJkPjwAAAAAAAACCAAAAFXBlcm1pdC1YMTEtZm9yd2FyZGluZwAAAAAAAAAXcGVybWl0LWFnZW50LWZvcndhcmRpbmcAAAAAAAAAFnBlcm1pdC1wb3J0LWZvcndhcmRpbmcAAAAAAAAACnBlcm1pdC1wdHkAAAAAAAAADnBlcm1pdC11c2VyLXJjAAAAAAAAAAAAAAEXAAAAB3NzaC1yc2EAAAADAQABAAABAQDBzOZVYg9fUyWGCJBLvWRMaW3BWexjNcPoQ1sOY2zHNliEnI9Ui87XpTTmxltbkToeBBZarBrNXlaFJsY1ZZ9WUh8HWo8T54iY8odNKzCS9MIuZkgT8l/LJzt54zI+XXXcwqEm0QvmRmnEsLmBC1Td7n2q0f87nFoTCFXHNd6EPdAi3G8IU2uxldbswu/xCp8yQw1OqjzSzt9jpuTyr10PQmV8uDu3H06IeUfQiFgCSI7kwumV9alIIP4rhhqz4nySF+mC83Y5r/AoKAD3rEPu/rfmVN5V6jt1wz3WMLWbIDI7FByDupCAcK8Tr901bAlh8mTBD5Y2VTB/luczebm1AAABDwAAAAdzc2gtcnNhAAABAJQbKtw8n2YJnQjezZ8Qv1ogi2GH+nUslkoYYhd0WptNxGDmaSCm/bdnmaQ3+9IczUe7Vieklw4aFYXfn2qeHFEKAEM92TKnxktivlS7tQok3WerKxMBKSCO0yLe3qHVVVGDBBtsz3RJvLnplDRQqA1CJ+IEgcWsHIIqKlVYQZDsb7fwH2Uf8EVN1E9V3BhxR6AcyfbyVCPegBKy/muQHNsOzDmmOaMQEDLLOZDp2sHLdQMSRXYeBhcAN7igbJvejVcbtznxnxl9iQwiDbuaVggSAj0TpplE8rE9G0fosowhiq4qztAPqaOnuKzhqnHQtI8RABnlJ7t8xrQmyriSakc=
```


`RSA-2048` 用户证书（1.17 版本服务端签发）：

```text
ssh-rsa-cert-v01@openssh.com AAAAHHNzaC1yc2EtY2VydC12MDFAb3BlbnNzaC5jb20AAAAgi/KmRy5YtOjKCUSAitf10h5IdWpACFUDo9NSP6qSs5wAAAADAQABAAAAgQDfGZzMFJ2SzSEAOon6/KXjqx2SRgS2KvBNTb4DI1Yxqa4pBSyJnCcTFNib/HR2JVwN7JleBQom3l176f1mhxC5mlmK36M0Q1/wpLxaS2WKRGXB/MSiaAZZ6GMstxjFbT3VRLCRCrXnNbuITuWjaME1s1C8pXz32VhRUTSmSQJY/QAAAAAAAAAAAAAAAQAAACIwIzIwMDUwMDA2OTIjcGFuZGF5Y2hlbiMxNjc5NjUwNDAzAAAACAAAAARyb290AAAAAGQPXmMAAAAAZCt+YwAAAAAAAACCAAAAFXBlcm1pdC1YMTEtZm9yd2FyZGluZwAAAAAAAAAXcGVybWl0LWFnZW50LWZvcndhcmRpbmcAAAAAAAAAFnBlcm1pdC1wb3J0LWZvcndhcmRpbmcAAAAAAAAACnBlcm1pdC1wdHkAAAAAAAAADnBlcm1pdC11c2VyLXJjAAAAAAAAAAAAAAEXAAAAB3NzaC1yc2EAAAADAQABAAABAQDEO4S/IT4G0A3qq2Ef+8oC4SDokRKDs3hiFK2cy7IXDVXZ7QJh69jRB2jgT/uZ/is931/K0MZzxTpTrd/10HJ1vYNTq1MBADgGk5Mn0Sak3EgHRdqvz18R8dRMMk+9GmZaY1vyljxxIeJPBUEX+KL+mdOD9p+jsjvpdd51GrJ2K8bP4mypM/djzuqLZpMgWVm2ag5G7MKvBZElj4aDX2fmp5aKvBGNNzMSLT7Oa6jOeDiZMtd1dxCyXD0fpJeHJtsiFIGhVgCfAI/sJj0E61vo7hOB6z/x4LesO2ppUCjTCGGWKB/t7UIRT8Rp8QGPh4Uj285J75QjpFFr1FRPGId9AAABFAAAAAxyc2Etc2hhMi0yNTYAAAEAodWzSWaumaululv6bgUpT/3A2Sl+EefVC9JcB6+oOybi26lg+V+wj/HcK2N5TW/Ov//xJGt6+fwNyOE6VFamzkzYAjlAn8OZxredW6xfnWDVsNh8JTHkKAwzgQ24V+U2Yrrl3iQ0S/BOTpMxdRusHyc37dTo1vVeTKbB7VDUKqAf5qxEUI3PtC8gxRDER68mKFHc+vy35vzYRqs/sHLTpYclokUlBmtcwrl8DLWCT1FWGz7iFmfHsvL3VIeGI+piKPg4IhVvaNKpwu8scrrbn1oHLq/CTuiR2h6OAJR3Mg+ULlemZhYsO72gzQ3Oz4rAya7Tftp5soIKH/Oxt2eNdQ==
```

| go 客户端版本 | go 证书服务版本（证书版本） | SSHD 版本 | 登录成功 | 报错 |
|  -------- |-------- |-------- |-------- |-------- |
| 1.11 | 1.11 | sshd6.0p1|succ | 无 |
| 1.11 | 1.17 | sshd6.0p1| failed | 见图 1.1 |
| 1.17| 1.11| sshd6.0p1|  succ | 无 |
| 1.17| 1.17| sshd6.0p1| failed| 见图 1.2 |
| 1.11 | 1.11 | sshd7.4p1|succ | 无 |
| 1.11 | 1.17 | sshd7.4p1| succ | 无 |
| 1.17| 1.11| sshd7.4p1|  failed | 见图 1.3 |
| 1.17| 1.17| sshd7.4p1| failed| 见图 1.4 |
| 1.11 | 1.11 | sshd8.8p1|failed | 见图1.7 |
| 1.11 | 1.17 | sshd8.8p1| failed | 见图1.8 |
| 1.17| 1.11| sshd8.8p1|  failed | 见图 1.9 |
| 1.17| 1.17| sshd8.8p1| succ| 无 |

不过，图1.9的错误，可以通过修改sshd_config配置来解决：
```TEXT
CASignatureAlgorithms ecdsa-sha2-nistp256,ecdsa-sha2-nistp384,ecdsa-sha2-nistp521,ssh-ed25519,rsa-sha2-512,rsa-sha2-256,ssh-rsa
```


图1.7/1.8的客户端的报错如下，这个暂时无解：
```text
[root@VM_130_14_centos ~]# ./client -addr x.x.x.x:36000
2023/03/31 13:08:15 dial error: ssh: handshake failed: ssh: no common algorithm for host key; client offered: [ssh-rsa-cert-v01@openssh.com ssh-dss-cert-v01@openssh.com ecdsa-sha2-nistp256-cert-v01@openssh.com ecdsa-sha2-nistp384-cert-v01@openssh.com ecdsa-sha2-nistp521-cert-v01@openssh.com ssh-ed25519-cert-v01@openssh.com ecdsa-sha2-nistp256 ecdsa-sha2-nistp384 ecdsa-sha2-nistp521 ssh-rsa ssh-dss ssh-ed25519], server offered: [rsa-sha2-512 rsa-sha2-256]
```

####    错误1.1
![PIC1](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/ssh/cert-bugs/pic1.1.png)

####    错误1.2
![PIC2](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/ssh/cert-bugs/pic1.2.png)

####    错误1.3
![PIC3](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/ssh/cert-bugs/pic1.3.png)

####    错误1.4
![PIC4](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/ssh/cert-bugs/pic1.4.png)

####    错误1.7
![PIC7](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/ssh/cert-bugs/pic1.7.png)

####    错误1.8
![PIC8](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/ssh/cert-bugs/pic1.8.png)

####    错误1.9
![PIC9](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/ssh/cert-bugs/pic1.9.png)


所以，结论是当用户证书使用 `RSA-2048` 时，`go1.11` 版本的兼容性是最好的

##      0x02    原因


##  0x03    总结

##  0x04    参考
-   [Use RSA CA Certificates with OpenSSH 8.2](https://ibug.io/blog/2020/04/ssh-8.2-rsa-ca/)
-   [SSH Certificates: How Do OpenSSH Certificates Compare to X.509?](https://goteleport.com/blog/x509-vs-openssh-certificates/)

转载请注明出处，本文采用 [CC4.0](http://creativecommons.org/licenses/by-nc-nd/4.0/) 协议授权
