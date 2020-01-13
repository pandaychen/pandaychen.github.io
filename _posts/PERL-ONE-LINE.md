---
layout:     post
title:      Perl单行技巧
subtitle:   
date:       2019-11-24
author:     pandaychen
catalog:    true
tags:
    - Perl
    - Perl单行命令
---

##  Perl
`Perl` 正则表达式正好是Perl的强项，很多问题都可以在命令行中执行单行Perl解决。本文简单介绍一下perl的单行执行的脚本，揭示一下perl的强大功能。
http://xiaoxuenotes.com/blog/2013/06/26/perl-oneline-command.html

http://highdb.com/perl-one-line-command-%E4%BB%8B%E7%BB%8D/

---------------------------------分割线---------------------------------------------
-e开关允许我们在命令行中运行脚本
1 perl -e 'print "hello\n"'
2 ls -lAF|perl -e 'while (<>) { next if /^dt/; $sum += (split)[4] } print "$sum\n"' ----取从ls中获得输入，找到文件大小的字段，计算所有文件大小的总和。
3 ls -lAF |perl -e 'while (<>){print $_}'  <==>等价于 ls -lAF |perl -ne 'print $_' -- 按行打印管道输入的结果
4 ls -lAF | perl -e 'while (<>) { next if /^[dt]/; print $_; }' 忽略以t或d开头的行
或者这样
perl -e 'while (<>) { print unless /\s+#/ }' < 1 -----从文件获取输入
不加<也行 perl -e 'while (<>) { print unless /\s+#/ }'  1

5 如何获取命令行参数
 perl -e 'print $_,"\n" for @ARGV' a b c 

cat files.txt | xargs wc -l ===> 任何时刻都不要忘记使用强大的xargs
Unix的xarg命令能够把他读取的数据转换成另一个命令的参数。我想把这些文件名转换成wc命令的参数以便于统计每个文件中行数。



--------------------------------------分割线---------------------------------------------
Perl 命令行程序既轻巧又便捷, 它特指单行代码的 Perl 程序, 处理一些事情会特别方便, 比如: 更改文本行的空白符, 行数统计, 转换文本, 删除和打印指定行以及日志分析等.
熟悉命令行操作后可以节省我们大量的时间成本, 当然了解 Perl 的基本语法和一些特殊符是学习 perl 命令行的基础. Perl 在 5.8 和 5.10 版本对命令行的支持都很好.
先来介绍下常用的参数：
   -a 参数在和 -n 或 -p 参数使用时, 启用自动分割模式, 将结果保存到数组@F(等同 @F = split $_ )
 -e 参数允许 Perl 代码以命令行的方式执行.(一定要加)
 -n 参数相当于在代码外围增加了while(<>)处理.(按行遍历)
   -p 参数等同-n参数, 不过会打印出行内容.（同上）
 -i 参数保证文件 file 在编辑之前被替换掉, 如果我们提供了扩展名, 则会对文件 file 做一个以扩展名结尾的备份文件.

   -M 参数保证在执行程序之前，加载指定的模块.
-l 参数自动chomp每行内容(去掉每行结尾的换行符)同时在打印输出的时候又加上换行符到结尾.（常用,l常和n搭配）
注意命令选项的次序!
所以:常用的perl单行命令为 perl -el   或者 perl -lne 或者 perl -nle(不打印每行内容)
我们以一些常用的示例开始说明:
一、比如,打印一行文件

可不能写成这样子(awk):下面这种是不行的

注意:e这个选项一定要放在最后!!!!!!!!!!!!!!!!!!!!!!!!,下面这两种都是错误的,没有输出


注意如下的例子,-e选项都是放在后面的
二、   perl -pi-e 's/from/to/g' file (文本替换,非按行)

该命令会替换文件file中所有的from为to, 如果要在windows中执行，将上述的 ‘ 换位 ” 即可.
因此很容易理解以下示例,先生成 file.bak 备份, 再进行替换操作:
   perl -pi.bak -e 's/from/tp/g' file

如果有多个文件, 则依次进行处理:
   perl -pi.bak -e 's/from/to/g' file1 file2 file3

在这里, 我们可以使用正则来只处理匹配到的行:
 perl -pi.bak -e 's/from/tp/g if /there/' file
将上面的命令行, 还原为 Perl 程序, 类似如下:
   while(<>) {
       if($_ =~ /there/) {
           $_ =~ s/from/to/g;
       }
       print $_;
   }

这里的正则可以是任何表达式, 比如我们只处理包含数字的行, 可以使用 \d 匹配数字:
   perl -pi -e 's/from/to/g if /\d/' file

同样我们也可以统计文本中相同行的信息, 打印超过一次的行(使用临时hash):
perl -ne 'print if $a{$_}++' file
这条命令使用了哈希 %a, 统计了一行内容出现的次数, 哈希的 key 为行的内容, 值为出现的次数; 在处理一行记录时, 如果 $a{$_} 为 0, 表示还没有处理过该行内容, 则忽略打印, 同时初始化 $a{$_} 赋值为 1; 如果 $a{$_} 大于 0, 则表示已经处理过该行, 这时满足 if 条件, 则打印出该行内容.
$_ 表示当前行的内容.
也可以使用 $. 打印行号, $. 变量维护着当前行号的信息, 只需要将其和 $_ 一起打印即可:
   perl -ne 'print "$. $_"' file

   perl -pe '$_ = "$. $_"' file

上面两个示例等效, 因为 -p 等同 -n, 同时也进行打印操作.结合上述统计文件相同行的示例, 只打印超过一次的行及行号:
   perl -ne 'print "$. $_" if $a{$_}++'

再来处理空白行, 如下示例:
   perl -lne 'print if length' file

如果不为空行则打印, 上述等同于 perl -ne ‘print unless /^$/’ file, 不过和下面示例有点不同:
   perl -ne 'print if /\S/'

\S正则匹配非空白字符(空格, 制表符, 新行, 回车等), 索引上述的两个示例并不会过滤制表符类的空行.
另一个例子我们采用 List::Util 模块( http://www.cpan.org )来打印每行中最大的数值，List::Util 是 Perl 的内置模块, 不需要额外的安装，以下
示例打印每行中最大的数值:
   perl -MList::Util=max -alne 'print max @F' file

-M 导入了 List::Util 模块, =max 导入了 List::Util 模块中的 max 方法, -a 开启了分割模式, 将结果存到 @F 数组中, -l 确保每次打印产生换行.
下面是一个随机生成8位字母组成的口令信息:
   perl -le 'print map{ ("a".."z")[rand 26] } 1..8'

“a”..”z” 生成字母从 a 到 z 的字母列表, 然后随机的选择 8 次.
我们可能也想知道一个 ip 地址对应的十进制整数:
   perl -le 'print unpack("N", 127.0.0.1)'

unpack 对应 pack 函数, N 表示32位网络地址形式的无符号整形, 详见 perldoc pack.
如果要进行计算该怎么做? -n 已经表示了 while(<>), 笨一点的方法可以将while放到代码里面, 比如以下:
   perl -e '$sum = 0; while(<>){ @f = split; $sum += $f[0]; } print $sum'

但是结合 -a 和 -n 选项则很容易处理:
   perl -lane '$sum += $F[0]; END{print $sum}'
END确保在程序即将结束时, 打印出总和.
再来看看统计iptables中通过iptables 规则的包的总量, 第一列显示了每条规则通过的包数, 我们只需要进行统计计算即可:
   iptables -nvxL | perl -lane '$pkt += $F[0]; END{print $pkt}'

最后介绍一点perldoc相关的信息, perldoc perlrun命令会显示如何执行 Perl 及命令行参数使用相关的文档信息, 这在我们忘记一些参数的时候非常有用; perldoc perlvar显示所有变量相关的文档信息, 一些记不住的特殊符总在这里能找到; perldoc perlop 显示了所有操作符相关的信息，应有尽有; perldoc perlfunc 显示所有函数的文档信息, 可以算得上函数大全了.

-a选项类似于awk的-F 选项
例如，文本如下

累加第三列的值并打印出来,使用-F作为分隔符录入选项(默认为空格或者\t)






##  Perl单行入门
        在工作和生活中难免有一些文件处理之类的工作需要做，不管你是在Windows下面，还是在Linux下面。当然稍微复杂一些的任务就需要利用正则表达式来完成，很不幸的是Linux shell的正则表达式不太好用，另外对正则表达式的支持也不够好，另外需要学习grep，sed，awk等命令也是挺烦的事儿。当然正则表达式正好是Perl的强项，很多问题都可以在命令行中执行单行Perl解决。本文简单介绍一下perl的单行执行的脚本，揭示一下perl的强大功能。
-e选项
     该选项表明后面跟随一条perl脚本。例如：
perl -e 'print "hello world!\n"'
将会在标准输出终端输出“hello world”。
-n选项
     该选项将会循环读入文件，但是不输入。例如：
该脚本相当于：
while(<>) { print $_; }
注意-n选项只是把文件循环读取一遍，具体的处理需要后面的脚本提供。
在例如打印/etc/passwd中包含root的行：
perl -ne 'print $_ if(/root/)' /etc/passwd-p选项
     该选项循环读取文件，并输出到标准输出。例如：
perl -pe 1
相当于：
while(<>) { 1; } continue { print $_; }
参数1没有实际意义，支持为了让perl语法正确。可以在print之前对当前行进行特殊处理，例如：
perl -pe '$_ ="Simon: " . $_'
将会将每行都以“Simon: ”开头。
-l 选项
     该选项将会使输入字符串去掉回车符号，而输出再加上回车符号。一般这个选项都需要加上，这样就可以组成-lpe和-lne两种组合。
-i选项
     该选项将会在文件中就地修改，并把源文件备份到参数指定后缀的文件中，默认情况下（不指定参数）为-i.bak，用户可以指定其他的后缀。
perl -i -pe 's/\r//g' file
该脚本的作用是将file从dos格式转换成Unix格式，源文件备份到file.bak中。
-a选项
     该选项将会以空格分割输入，并把分割结果存放到@F数组中。
$ echo a b c | perl -lane 'print $F[1]' b
 $ echo a b c | perl -lane 'print "@F[0..1]"' a b 
$ echo a b c | perl -lane 'print "@F[-2,-1]"' b c
再例如：显示/etc/passwd文件大小，我们首先通过ls -l显示文件的详细信息，发现第五列就是文件大小。则用如下脚本即可实现：
ls -l /etc/passwd | perl -lane 'print $F[4]'
-F选项
     该选项可以让你指定分割符，以代替a选项的空白分割符，注意该选项需要和a选项一起使用。例如需要查找系统中的所有用户：
perl -F: -lane 'print $F[0]' /etc/passwd
该命令相当于如下代码：
while(<>) { @F = split \:\; print $F[0] . "\n" }
     如果熟练使用如上选项，并且对perl比较熟悉的话，基本就可以抛弃shell的正则表达式的依赖了，处理问题也不需要去查grep、awk、sed等的用法了。
  
文二



##  Perl单行提高

替换
将所有C程序中的foo替换成bar，旧文件备份成.bak
perl -p -i.bak -e 's/\bfoo\b/bar/g' *.c
很强大的功能，特别是在大程序中做重构。记得只有在UltraEdit用过。 如果你不想备份，就直接写成 perl -p -i -e 或者更简单 perl -pie， 恩，pie这个单词不错
将每个文件中出现的数值都加一
perl -i.bak -pe 's/(\d+)/ 1 + $1 /ge' file1 file2 ....
将换行符\r\n替换成\n
perl -pie 's/\r\n/\n/g' file
同dos2unix命令。
将换行符\n替换成\r\n
perl -pie 's/\n/\r\n/g' file
同unix2dos命令。
取出文件的一部分
显示字段0-4和字段6，字段的分隔符是空格
perl -lane 'print "@F[0..4] $F[6]"' file
很好很强大，同 awk 'print $1, $2, $3, $4, $5, $7'。参数名称lane也很好记。
如果字段分隔符不是空格而是冒号，则用    #注意这个分隔符
perl -F: -lane 'print "@F[0..4]\n"' /etc/passwd
显示START和END之间的部分
perl -ne 'print if /^START$/ .. /^END$/' file
恐怕这个操作只有sed才做得到了吧……
相反，不显示START和END之间的部分
perl -ne 'print unless /^START$/ .. /^END$/' file
显示开头50行：
perl -pe 'exit if $. > 50' file
同命令 head -n 50
不显示开头10行：
perl -ne 'print unless 1 .. 10' file
显示15行到17行：
perl -ne 'print if 15 .. 17' file
每行取前80个字符：
perl -lne 'print substr($_, 0, 80) = ""' file
每行丢弃前10个字符：
perl -lne 'print substr($_, 10) = ""' file
搜索
查找comment字符串：
perl -ne 'print if /comment/' duptext
这个就是普通的grep命令了。
查找不含comment字符串的行：
perl -ne 'print unless /comment/' duptext
反向的grep，即grep -v。
查找包含comment或apple的行：
perl -ne 'print if /comment/ || /apple/' duptext
相同的功能就要用到egrep了，语法比较复杂，我不会……
计算
计算字段4和倒数第二字段之和：
perl -lane 'print $F[4] + $F[-2]'
要是用awk，就得写成 awk '{i=NF-1;print $5+$i}'
排序和反转
文件按行排序：
perl -e 'print sort <>' file
相当于简单的sort命令。
文件按段落排序：
perl -00 -e 'print sort <>' file
多个文件按文件内容排序，并返回合并后的文件：
perl -0777 -e 'print sort <>' file1 file2
文件按行反转：
perl -e 'print reverse <>' file1
相应的命令有吗？有……不过挺偏，tac（cat的反转）
数值计算
10进制转16进制：
perl -ne 'printf "%x\n",$_'
10进制转8进制： perl -ne 'printf "%o\n",$_'
16进制转10进制：
perl -ne 'print hex($_)."\n"'
8进制转10进制：
perl -ne 'print oct($_)."\n"'
简易计算器。
perl -ne 'print eval($_)."\n"'
其他
启动交互式perl：
perl -de 1
查看包含路径的内容：
perl -le 'print for @INC'
备注
与One-Liner相关的Perl命令行参数：
-0<数字> (用8进制表示)指定记录分隔符($/变量)，默认为换行 -00 段落模式，即以连续换行为分隔符 -0777 禁用分隔符，即将整个文件作为一个记录 -a 自动分隔模式，用空格分隔$_并保存到@F中。相当于@F = split ''。分隔符可以使用-F参数指定 -F 指定-a的分隔符，可以使用正则表达式 -e 执行指定的脚本。 -i<扩展名> 原地替换文件，并将旧文件用指定的扩展名备份。不指定扩展名则不备份。 -l 对输入内容自动chomp，对输出内容自动添加换行 -n 自动循环，相当于 while(<>) { 脚本; } -p 自动循环+自动输出，相当于 while(<>) { 脚本; print; }
1-->显示历史命令使用频率
criver@ubuntu:~$ history | perl -F"\||<\(|;|\`|\\$\(" -alne 'foreach (@F) { print $1 if /\b((?!do)[a-z]+)\b/i }' | sort | uniq -c | sort -nr
    169 ls
     98 vim
     90 python
     27 man
 
2-->将每个文件中出现的数值都加一
perl -i.bak -pe 's/(\d+)/ 1 + $1 /ge' file1 file2 ....
criver@ubuntu:~$ cat ptt1.txt
204.108.13.15 abc [] ServerPath=/home/html/pics 62ms
214.92.113.13 xxx [code=5] ServerPath=/home/html/pages 32ms
criver@ubuntu:~$ perl -i.bak -pe 's/(\d+)/ 1 + $1 /ge' ptt1.txt    
匹配servrerpath后的字符并打印出来
criver@ubuntu:~$ perl -ne  'print "$1\n" if  /ServerPath=(\S+)/g' ptt1.txt
/home/html/pics
/home/html/pages
 
give starstar 3-->查看包含路径的内容：
perl -le 'print for @INC'    等价于 perl -le 'print $_ for @INC'
这样也可以
perl -le 'for $a (@INC){print $a}'    ---->这里就是完整的perl代码了

4-->取出文件的一部分
显示字段0-4和字段6，字段的分隔符是空格
perl -lane 'print "@F[0..4] $F[6]"' file
很好很强大，同 awk ‘print $1, $2, $3, $4, $5, $7′。参数名称lane也很好记。
如果字段分隔符不是空格而是冒号，则用
perl -F: -lane 'print "@F[0..4]\n"' /etc/passwd
显示START和END之间的部分
perl -ne 'print if /^START$/ .. /^END$/' file
恐怕这个操作只有sed才做得到了吧……
相反，不显示START和END之间的部分
perl -ne 'print unless /^START$/ .. /^END$/' file
显示开头50行：
perl -pe 'exit if $. > 50' file
同命令 head -n 50
不显示开头10行：
perl -ne 'print unless 1 .. 10' file
显示15行到17行：
perl -ne 'print if 15 .. 17' file
每行取前80个字符：
perl -lne 'print substr($_, 0, 80) = ""' file
每行丢弃前10个字符：
perl -lne 'print substr($_, 10) = ""' file
 
5-->准备关键词测试时经常需要进行UrlEncode和UrlDecode
以下是2个常用的单行Perl脚本（正则表达式）：输入为日志或关键词列表
Urlencode：对 \n 不转码
perl -p -e 's/([^\w\-\.\@])/$1 eq "\n" ? "\n":sprintf("%%%2.2x",ord($1))/eg' keywords.list
UrlDecode：
perl -p -e 's/%(..)/pack("c", hex($1))/eg' query.log
 
备注
与One-Liner相关的Perl命令行参数

-0<数字>
    (用8进制表示)指定记录分隔符($/变量)，默认为换行
-00
    段落模式，即以连续换行为分隔符
-0777
    禁用分隔符，即将整个文件作为一个记录
-a
    自动分隔模式，用空格分隔$_并保存到@F中。相当于@F = split ”。分隔符可以使用-F参数指定
-F
    指定-a的分隔符，可以使用正则表达式
-e
    执行指定的脚本。
-i<扩展名>
    原地替换文件，并将旧文件用指定的扩展名备份。不指定扩展名则不备份。
-l
    对输入内容自动chomp，对输出内容自动添加换行
-n
    自动循环，相当于 while(<>) { 脚本; }
-p
    自动循环+自动输出，相当于 while(<>) { 脚本; print; }
