---
layout:     post
title:      玩转正则表达式（进阶篇）
subtitle:   贪婪 or 非贪婪，Python 版本
date:       2020-01-12
author:     pandaychen
catalog:    true
tags:
    - 正则表达式
---

##  0x00    前言
&emsp;&emsp; 本文介绍一些正则表达式 [（Regular Expression）](https://zh.wikipedia.org/wiki/%E6%AD%A3%E5%88%99%E8%A1%A8%E8%BE%BE%E5%BC%8F) 一些很有趣（实用）的语法。

## 0x01 问号的用法（?）
&emsp;&emsp; 来看看 Wikipedia 中对问号（?）的解释：
>   非贪心量化（Non-greedy quantifiers）：当该字符紧跟在任何一个其他重复修饰符（*,+,?，{n}，{n,}，{n,m}）后面时，匹配模式是非贪婪的。非贪婪模式尽可能少的匹配所搜索的字符串，而默认的贪婪模式则尽可能多的匹配所搜索的字符串。例如，对于字符串 "oooo"，"o+?" 将匹配单个 "o"，而 "o+" 将匹配所有 "o"。

####	基础用法
作为数量词，用在字符或 (...) 之后，匹配前一个字符 `0` 次或 `1` 次

```python
qm_re1=re.compile(r'abcc?')
str1="abc"
print qm_re1.match(str1).group()

qm_re2=re.compile(r'def(abc)?')
str2="defhijkabc"
print qm_re2.match(str2).group()
```

####   ? 开头的组合
以问号开头的表达式，一定是和圆括号 `(pattern)` 的组合，但并非是分组 `(pattern)` 的概念。 常见有：`(?:pattern)`，`(?<=pattern)`，`(?!pattern)` 及 `(?>!pattern)`

1、`(?:pattern)` 模式<br>
该模式是针对于 `(pattern)` 而言的，就是你这里需要加括号但是不希望它成为一个分组
> (pattern)	匹配 pattern 并获取这一匹配的子字符串。该子字符串用于向后引用。所获取的匹配可以从产生的 Matches 集合得到，在 VBScript 中使用 SubMatches 集合，在 JScript 中则使用 $0…$9 属性。要匹配圆括号字符，请使用 "\(" 或 "\)"。可带数量后缀。
> 匹配 pattern 但不获取匹配的子字符串（shy groups），也就是说这是一个非获取匹配，不存储匹配的子字符串用于向后引用。这在使用或字符 "(|)" 来组合一个模式的各个部分是很有用。例如 `industr(?:y|ies)` 就是一个比 "industry|industries" 更简略的表达式。

例如：
```python
qm_re3=re.compile(r'^(\w+)(?:,|#)(\w+)$')
str22="industry#industries"
print qm_re3.match(str22).groups()
```

2、`(?=pattern)` 模式<br>
此模式 `(?=pattern)` 表示匹配以 `pattern` 结尾的前面部分（匹配到的内容不包含 `pattern`），一般是加在所要匹配内容的后面。如下所示：
```python
qm_re5=re.compile(r'(\w+)(?=xyz)')
str23="abcdefghijkxyz"
print qm_re5.match(str23).group()
```
输出为 `abcdefghijk`，匹配到了 `xyz` 之前所有的单词字符 `[A-Za-z0-9]`

3、`(?<=pattern)` 模式<br>
此模式 `(?<=pattern)` 表示匹配以 `pattern` 结尾的后面部分（匹配到的内容不包含 `pattern`）, 一般是加在所要匹配的内容前面。

```python
str23="abcdefghijkxyz"
qm_re6=re.compile('(?<=abc)(\w+)')
print re.search(qm_re6,str23).groups()
```
输出为 `('defghijkxyz',)`

4、`(?!pattern)` 模式<br>
`(?!pattern)` 就是匹配后面不跟 "pattern" 的

5、`(?>!pattern)` 模式<br>
`(?>!pattern)` 就是匹配前面不跟 "pattern" 的


##	0x03    Python 中 `re.search()` 和 `re.match()` 等的区别
-   `match`
匹配 string 开头，成功返回 Match object, 失败返回 None ，只匹配一个。
-   `search`
在 string 中进行搜索，成功返回 Match object, 失败返回 None , 只匹配一个。
-   `findall`
在 string 中查找所有匹配成功的分组。返回 `list` 对象，每个 item 是由每个匹配的捕获分组组成的 list。
-   `finditer`
在 string 中查找所有匹配成功的字符串, 返回 `iterator`，每个 item 是一个 Match object


##  0x04    不建议使用 `re.compile()`

##  0x05    参考
-   [Python 正则表达式指南](https://www.cnblogs.com/huxi/archive/2010/07/04/1771073.html)
-	[细说正则表达式](https://www.jianshu.com/p/147fab022566)
-   [What is the difference between Python's re.search and re.match?](http://stackoverflow.com/questions/180986/what-is-the-difference-between-pythons-re-search-and-re-match)
-   [一日一技：请不要再用 re.compile 了](https://mp.weixin.qq.com/s?__biz=MzI2MzEwNTY3OQ%3D%3D&mid=2648977413&idx=1&sn=705b2055e7cd4b2caaf94eb67f236315&chksm=f2506de5c527e4f39159139a806e3f2b43b0682831c3e1caa26f8ad2b51ad59cf82b9ecf8968&scene=27)
-   [Details of the Cloudflare outage on July 2, 2019](https://blog.cloudflare.com/details-of-the-cloudflare-outage-on-july-2-2019/)

转载请注明出处，本文采用 [CC4.0](http://creativecommons.org/licenses/by-nc-nd/4.0/) 协议授权
