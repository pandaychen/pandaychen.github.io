---
layout:     post
title:      `Perl`单行技巧
subtitle:   
date:       2020-01-12
author:     pandaychen
catalog:    true
tags:
    - Perl
---

##  Perl
&emsp;&emsp;在日常处理文本时，`Perl`的单行模式非常有用。本文介绍下强大的单行特技。

## 
> -e开关允许我们在命令行中运行脚本
```perl
#简单的打印
perl -e 'print "hello\n"'
#取从ls中获得输入，找到文件大小的字段，计算所有文件大小的总和
ls -lAF|perl -e 'while (<>) { next if /^dt/; $sum += (split)[4] } print "$sum\n"'
#按行打印管道输入的结果
ls -lAF |perl -e 'while (<>){print $_}'  <==>等价于 ls -lAF |perl -ne 'print $_' 
#过滤以d或t开头的行
ls -lAF | perl -e 'while (<>) { next if /^[dt]/; print $_; }'
```

## 参考
-  [Introduction to Perl one-liners](https://catonmat.net/introduction-to-perl-one-liners)