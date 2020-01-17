---
layout:     post
title:      Perl单行特技
subtitle:   拾起了我心爱的小perl...perl
date:       2020-01-04
author:     pandaychen
catalog:    true
tags:
    - Perl
---

##  0x00 前言
&emsp;&emsp;在日常处理文本时，`Perl`的单行模式非常有用。本文介绍下强大的单行特技。

## 0x01	Perl单行应用
> `-e`开关允许我们在命令行中运行脚本

```perl
#简单的打印
perl -e 'print "hello\n"'
#取从ls中获得输入，找到文件大小的字段，计算所有文件大小的总和
ls -lAF|perl -e 'while (<>) { next if /^dt/; $sum += (split)[4] } print "$sum\n"'
#按行打印管道输入的结果
ls -lAF |perl -e 'while (<>){print $_}'
#过滤以d或t开头的行
ls -lAF | perl -e 'while (<>) { next if /^[dt]/; print $_; }'
```

> `-n`选项在你的程序中封装一个while循环，循环通过钻石操作符读取输入，把$_设置为读取的内容

```perl
如上面的
ls -lAF |perl -e 'while (<>){print $_}'
可以改为
ls -lAF |perl -ne 'print $_'
```

> `-p`选项做同样的事，并且每次递归后输出$_的值

```perl
ls -l | perl -pe 's/\S+ //'
```

## 参考
-  [Introduction to Perl one-liners](https://catonmat.net/introduction-to-perl-one-liners)
