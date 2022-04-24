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

| 按键 | name | ASCII-HEX |
| :-----:| :----: | :----: |
| ENTER| 回车	| 0x03 |


## 	0x02 CTRL 类指令
本小节仅针对 `/bin/bash` 的行为：
不区分大小写：
-	`CTRL`+`A`：当前光标移动到首位
![img](![index1](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/2022/bash/ctrl-a.png)
-	`CTRL`+`B`：当前光标向前移动 `1` 位（直至首位）
-	`CTRL`+`C`：取消当前行指令
![img]()
-	`CTRL`+`D`：从当前光标开始，向后依次删除 `1` 个字符；如果当前行无字符，直接退出当前会话
-	`CTRL`+`W`：清空当前行 buffer
-	`CTRL`+`R`：reverse-search
-	`CTRL`+`T`：有趣：当前光标前的两个位置字符位置相互调换
-	`CTRL`+`U`：情况当前行 buffer，看起来和 `W` 一样
-	`CTRL`+`I`：类似于 `Tab` 补全的功能
-	`CTRL`+`O`：类似于 `Enter` 的功能
-	`CTRL`+`P`：类似于上箭头 `UPARROW` 的功能
-	`CTRL`+`F`：从当前光标开始，依次向后移动 `1` 个位置
-	`CTRL`+`H`：从当前光标开始，依次向前删除 `1` 个字符
-	`CTRL`+`J`：类似于 `Enter` 的功能
-	`CTRL`+`K`：删除当前光标位置（包含此位置的字符），之后所有的字符
-	`CTRL`+`L`：清屏
-	`CTRL`+`X`：光标的当前位置移动到行首，再次输入，恢复到先前的位置
-	`CTRL`+`V`：复制（插入）缓冲区的内容到当前光标处
-	`CTRL`+`M`：类似于 `Enter` 的功能

##  0x03  参考
- [ascii](https://zh.wikipedia.org/zh/ASCII)