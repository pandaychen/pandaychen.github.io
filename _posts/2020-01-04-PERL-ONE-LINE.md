---
layout:     post
title:      Perl单行特技（One-Line）
subtitle:   拾起了我心爱的小perl...perl
date:       2020-01-04
author:     pandaychen
catalog:    true
tags:
    - Perl
---

##  0x00 前言
&emsp;&emsp;在日常处理文本时，`Perl`的单行模式非常有用。本文介绍下强大的单行特技。

与`One-Line`相关的参数
>   -a 自动分隔模式，用空格分隔$并保存在@F中，也就是@F=split //, $
>   -F 指定-a的分隔符
>   -l 对输入的内容进行自动chomp，对输出的内容自动加换行符
>   -n 相当于while(<>)
>   -e 执行命令，也就是脚本
>   -p 自动循环+输出，也就是while(<>){命令（脚本）; print;}

![image](https://s2.ax1x.com/2020/01/17/lzsUsJ.png)

## 0x01	Perl单行选项
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
#如上面的
ls -lAF |perl -e 'while (<>){print $_}'
#可以改为
ls -lAF |perl -ne 'print $_'
```

> `-p`选项做同样的事，并且每次递归后输出$_的值

```perl
ls -l | perl -pe 's/\S+ //'
```

> `-l` 对输入的内容进行自动`chomp`（去换行符），对输出的内容自动加换行符

##  0x02    处理文件的快捷操作

1. 在多个文件中处理替换
```perl
perl -pi -e 's/test1/test2/g' file1 file2 file3
```
2.  处理文件时匹配指定的行，然后进行替换
```perl
#把file文件中包含hello这行中的test1全部替换成test2
perl -pi -e 's/test1/test2/g if /hello/' file
#把file文件中匹配到包含数字的行全部把test2替换成test1
perl -pi -e 's/test1/test2/g if /\d/' file
```

3. 文件的行去重（hash）
```perl
perl -ne 'print if $a{$_}++' file
```

4. 打印行号
```perl
perl -ne 'print "$. $_"'
#使用-p的方式实现，修改默认的$_变量
perl -pe '$_ = "$. $_"'
```

5.	输出重复行，并且打印重复行的行号
```perl
perl -ne 'print "$. $_" if $a{$_}++' file
```

6.	`perl -pe` 可用于读取文件每行，并按照给定的命令进行处理，最后输出
```perl
perl -pe 's/aaa/AAA/g' file
```
	> 注意：使用`-p`选项比`-n`更为直观一些

7.	`perl -l`参数几乎可以跟`-n`搭配代替perl经常用的`while(<>){chomp;}`语法

8.	重点!如果需要处理tab分割的文件的每一行内容，那么`perl -alne`参数几乎可以说是必备的
例如，
```perl
while(<>){
	chomp;
	@F=split /\s+/,$_;
	print "$F[0]\n"
}
```
上面这段代码相当于`perl -alne 'print $F[0]'`
9.	其他一些例子

```perl
#输入数字的行，使用正则匹配
perl -ne 'print if /^\d+$/'	

#perl单行命令脚本里的变量都不需要预先声明，如想打印出每空行，并且每行以行数开头
perl -ne 'print ++$a." $_" if /./'

#perl单行命令有时优于sed/grep等shell命令是由于其优秀的正则匹配，通常简单的匹配可以如：匹配上的行号，模仿grep -c的功能
perl -lne '$a++ if /regex/; END {print $a+0}'

#perl单行命令可以使用perl的模块，如使用sum函数的模块
perl -MList::Util=sum -alne 'print sum @F'

#perl也可以像awk一样使用END命令，如打印出文件中总单词个数
perl -alne '$t += @F; END { print $t}'

#perl也可以使用map{}等函数，如打印出匹配上的单词的总个数
perl -alne 'map { /regex/ && $t++ } @F; END { print $t }'

#perl单行命令可以说是将perl的简洁用到了极致，如打印出匹配上的行：
perl -ne '/regex/ && print'
```

## 参考
-  [Introduction to Perl one-liners](https://catonmat.net/introduction-to-perl-one-liners)
