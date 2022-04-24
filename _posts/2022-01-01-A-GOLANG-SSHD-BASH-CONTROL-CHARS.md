---
layout: post
title: 使用 Golang 实现 SSH 和 SSHD（三）
subtitle: Bash 中的控制字符功能说明
date: 2022-01-01
author: pandaychen
catalog: true
tags:
  - OpenSSH
---

## 0x00 前言
项目中，需要实现 OpenSSH 字符终端审计及输入命令还原，因而需要了解 Bash 中的特殊按键行为及其对终端屏显的影响。本篇文章梳理下特殊按键的行为。


##	0x01	常用按键对应的 ascii 码
| 按键 | name | ASCII(HEX) |
| :-----:| :----: | :----: |
| ENTER| 回车	| 0x03 |


## 	0x02 CTRL 类指令
不区分大小写：
-	`CTRL`+`A`：当前光标移动到首位
![img](![index1](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/2022/bash/ctrl-a.png)
-	`CTRL`+`B`：当前光标向前移动 `1` 位（直至首位）
-	`CTRL`+`C`：取消当前行指令

##  0x03  参考
- [ascii](https://zh.wikipedia.org/zh/ASCII)