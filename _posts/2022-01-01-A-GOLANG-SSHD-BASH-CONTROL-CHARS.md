---
layout: post
title: 使用 Golang 实现 SSH 和 SSHD（三）
subtitle: Bash 中的控制字符功能说明 && 分析
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
-	`CTRL`+`A`：当前光标移动到首位（移到命令行首）
![img](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/2022/bash/ctrl-a.png)
-	`CTRL`+`B`：当前光标向前移动 `1` 位 / 个字符（直至首位）（按字符移动（左向））
-	`CTRL`+`C`：取消当前行指令，停止当前运行的命令。如果一个命令运行时间过久，或者你误运行了，可以使用本指令来强制停止或退出
![img](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/2022/bash/ctrl-c.png)
-	`CTRL`+`D`：从当前光标开始，向后依次删除 `1` 个字符，即删除光标后的一个字符；如果当前行无字符，直接退出当前会话（和 `exit` 指令类似）
-	`CTRL`+`W`：删除光标前的一个单词（注意，和 `CTRL`+`U` 不一样，`CTRL`+`W` 不会删除光标前的所有东西，而是只删除一个单词）。如下面的例子，输入为 `ifconfig eth1`，光标在 `h` 处，本指令操作结果如下：
![img1](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/2022/bash/ctrl-w1.png)
![img2](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/2022/bash/ctrl-w2.png)
- `CTRL`+`E`：移到当前行命令行尾（移到命令行尾），和 `END` 键功能类似
-	`CTRL`+`R`：reverse-search
-	`CTRL`+`T`：有趣：当前光标前的两个位置字符位置相互调换（交换光标处和之前的字符，交换最后两个字符）
-	`CTRL`+`U`：删除光标前的所有字符（从光标后的点删除到行首），该指令立刻删除前面的所有字符。和 `W` 不一样的在于，本指令从光标处删除至命令行首
-	`CTRL`+`I`：类似于 `Tab` 补全的功能
-	`CTRL`+`O`：类似于 `Enter` 的功能，执行当前命令，并选择上一条命令
-	`CTRL`+`P`：类似于上箭头 `UPARROW` 的功能
-	`CTRL`+`F`：从当前光标开始，依次向后移动 `1` 个位置（`B` 的反向）（按字符移动（右向））
-	`CTRL`+`H`：从当前光标开始，依次向前删除 `1` 个字符，即删除光标前的一个字符，和退格键的功能相同
-	`CTRL`+`J`：类似于 `Enter` 的功能
-	`CTRL`+`K`：删除当前光标位置（包含此位置的字符），之后所有的字符（`U` 的反向）。即删除光标后的所有字符
-	`CTRL`+`L`：清屏
-	`CTRL`+`X`：光标的当前位置移动到行首，再次输入，恢复到先前的位置（在命令行首和光标之间移动）
-	`CTRL`+`V`：复制（插入）缓冲区的内容到当前光标处
-	`CTRL`+`M`：类似于 `Enter` 的功能
- `CTRL`+`Y`：恢复上一个删除或剪切的条目。比如使用 `CTRL`+`W` 删除了单词 `eth1`。你可以使用 `CTRL`+`Y` 立刻恢复

####  其他
- `CTRL`+`Z`：停止当前的命令，即终止了当前运行的命令。可以在前台使用 `fg` 或在后台使用 `bg` 来恢复
- `CTRL`+`[`：和 `ESC` 键等同
- `CTRL`+`G`：退出历史搜索模式，不运行命令
- `CTRL`+`O`：运行使用反向搜索时发现的命令，即 `CTRL`+`R`
- `CTRL`+`R`：向后搜索历史记录（使用反向搜索时）
- `CTRL`+`S`：向前搜索历史记录

####  有意思的点
- `CTRL`+`J`：和 `ENTER`/`RETURN` 键相同。`CTRL`+`J` 或 `CTRL`+`M` 可以用来替换回车键
- `CTRL`+`N`：在命令历史中显示下一行，等同于下箭头键 `DownArrow`
- `CTRL`+`B`：光标向前移动一个字符。等同于左箭头键 `LeftArrow`
- `CTRL`+`F`：光标向后移动一个字符，等同于右箭头键 `RightArrow`
- `CTRL`+`P`：显示命令历史的上一条命令，等同于上箭头键 `UpArrow`

##  0x03  ALT 类命令
- `ALT`+`F`：按单词后移（右向）
- `ALT`+`B`：按单词前移（左向）
![img](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/2022/bash/alt-b.png)
- `ALT` + `D` ：从光标处删除至单词尾，不一定是行尾
- `ALT` + `C` ：从光标处更改为首字母大写的单词，并且光标移动到单词尾部
- `ALT` + `U` ：从光标处更改为全部大写的单词，并且光标移动到单词尾部
- `ALT` + `L` ：从光标处更改为全部小写的单词，并且光标移动到单词尾部
- `ALT` + `T` ：交换光标处和之前位置的两个单词
- `ALT` + `Backspace`：与 `CTRL` + `W` 类似，分隔符有些差别

##  0x04  Terminal input sequences

```text
<char>                                         -> char
<esc> <nochar>                                 -> esc
<esc> <esc>                                    -> esc
<esc> <char>                                   -> Alt-keypress or keycode sequence
<esc> '[' <nochar>                             -> Alt-[
<esc> '[' (<modifier>) <char>                  -> keycode sequence, <modifier> is a decimal number and defaults to 1 (xterm)
<esc> '[' (<keycode>) (';'<modifier>) '~'      -> keycode sequence, <keycode> and <modifier> are decimal numbers and default to 1 (vt)
```

```text
vt sequences:
<esc>[1~    - Home        <esc>[16~   -             <esc>[31~   - F17
<esc>[2~    - Insert      <esc>[17~   - F6          <esc>[32~   - F18
<esc>[3~    - Delete      <esc>[18~   - F7          <esc>[33~   - F19
<esc>[4~    - End         <esc>[19~   - F8          <esc>[34~   - F20
<esc>[5~    - PgUp        <esc>[20~   - F9          <esc>[35~   - 
<esc>[6~    - PgDn        <esc>[21~   - F10         
<esc>[7~    - Home        <esc>[22~   -             
<esc>[8~    - End         <esc>[23~   - F11         
<esc>[9~    -             <esc>[24~   - F12         
<esc>[10~   - F0          <esc>[25~   - F13         
<esc>[11~   - F1          <esc>[26~   - F14         
<esc>[12~   - F2          <esc>[27~   -             
<esc>[13~   - F3          <esc>[28~   - F15         
<esc>[14~   - F4          <esc>[29~   - F16         
<esc>[15~   - F5          <esc>[30~   -

xterm sequences:
<esc>[A     - Up          <esc>[K     -             <esc>[U     -
<esc>[B     - Down        <esc>[L     -             <esc>[V     -
<esc>[C     - Right       <esc>[M     -             <esc>[W     -
<esc>[D     - Left        <esc>[N     -             <esc>[X     -
<esc>[E     -             <esc>[O     -             <esc>[Y     -
<esc>[F     - End         <esc>[1P    - F1          <esc>[Z     -
<esc>[G     - Keypad 5    <esc>[1Q    - F2       
<esc>[H     - Home        <esc>[1R    - F3       
<esc>[I     -             <esc>[1S    - F4       
<esc>[J     -             <esc>[T     - 
```

####  输出文本
- 如`F5`，vt sequence为`<esc>[15~`，在golang中会显示为：`[27,91,49,53,126]`
- 如`UpArrow`，xterm sequence为`<esc>[A`，golang显示为：`[27,91,65]`


##  0x05  模拟键盘操作的状态机实现

##  0x06  VT100 commands and control sequences
`vt100`是一个非常常用的终端类型，`vt100`控制码是用来在在终端显示的代码，比如在终端上任意坐标用不同的颜色显示字符；其中所有的控制符都是由`\033`打头（`8`进制、即`ESC`的`ASCII`码），格式分下列两种：
- 数字形式，如`\033``[``<数字>``m` 
- 控制字符形式，如`\033``[``字母`

####  控制码
```text
\033[0m		// 关闭所有属性
\033[1m		// 设置为高亮
\033[4m		// 下划线
\033[5m		// 闪烁
\033[7m		// 反显
\033[8m		// 消隐
\033[nA		// 光标上移 n 行
\033[nB		// 光标下移 n 行
\033[nC		// 光标右移 n 行
\033[nD		// 光标左移 n 行
\033[y;xH	// 设置光标位置
\033[2J		// 清屏
\033[K		// 清除从光标到行尾的内容
\033[s		// 保存光标位置
\033[u		// 恢复光标位置
\033[?25l	// 隐藏光标
\033[?25h	// 显示光标
```

####  控制码-颜色类
`\033[30m – \033[37m` 为设置前景色，`\033[40m – \033[47m` 为设置背景色
```text
\033[30m ---- \033[37m   //设置前景色，0-7为 黑 红 绿 黄 蓝 紫 青 白
\033[40m ---- \033[47m   //设置背景色，0-7为 黑 红 绿 黄 蓝 紫 青 白
```

更详细的指令请参考[此文](http://www.braun-home.net/michael/info/misc/VT100_commands.htm)




##  0x07  关于终端的其他细节

####  终端的区别：vt100 VS xterm
终端类型，`vt100`/`vt102`/`vt220`/`xterm`直接的区别是什么？引用此问题[What's the difference between various $TERM variables?](https://unix.stackexchange.com/questions/43945/whats-the-difference-between-various-term-variables)

`xterm` is supposed to be a superset of `vt220`, in other words it's like `vt220` but has more features. For example, `xterm` usually supports colors, but `vt220` doesn't. You can test this by pressing `z` inside `top`.

In the same way, `vt220` has more features than `vt100`. For example, `vt100` doesn't seem to support `F11` and `F12`.

Compare their features and escape sequences that your system thinks they have by running infocmp <term type 1> <term type 2>, e.g. infocmp `vt100` `vt220.`

The full list varies from system to system. You should be able to get the list using toe, toe `/usr/share/terminfo`, or find `${TERMINFO:-/usr/share/terminfo}`. If none of those work, you could also look at ncurses' terminfo.src, which is where most distributions get the data from these days.

But unless your terminal looks like this or this, there's only a few others you might want to use:

- `xterm-color` - if you're on an older system and colors don't work
- `putty`, konsole, Eterm, rxvt, gnome, etc. - if you're running an XTerm emulator and some of the function keys, Backspace, Delete, Home, and End don't work properly
- `screen` - if running inside GNU screen (or tmux)
- `linux` - when logging in via a Linux console (e.g. Ctrl+Alt+F1)
- `dumb` - when everything is broken

##  0x09  zmodem协议

本小节分析下`rz/sz`的底层zmodem协议的基础知识

####  zmodem协议格式

Zmodem协议的格式包括以下几个部分：
1.  发送方发送起始帧：发送方发送起始帧，告诉接收方它要开始传输文件了
2.  接收方发送应答帧：接收方收到起始帧后，发送应答帧，告诉发送方它已经准备好接收文件了
3.  发送方发送文件头：发送方发送文件头，包括文件名、文件大小、修改时间等信息
4.  接收方发送应答帧：接收方收到文件头后，发送应答帧，告诉发送方它已经准备好接收文件数据了
5.  发送方发送文件数据：发送方发送文件数据，每发送一定数量的数据，就会等待接收方发送确认帧
6.  接收方发送确认帧：接收方收到文件数据后，发送确认帧，告诉发送方它已经成功接收了数据
7.  发送方发送结束帧：发送方发送结束帧，告诉接收方文件传输已经完成
8.  接收方发送应答帧：接收方收到结束帧后，发送应答帧，告诉发送方它已经成功接收了文件

在传输过程中，如果出现错误或中断，Zmodem协议可以从中断的地方继续传输，而不需要重新开始。这是通过发送方和接收方之间的握手协议来实现的。如果出现错误或中断，发送方会发送一个中断帧，接收方会发送一个中断应答帧，然后重新开始传输

##  0x0A 参考
- [ascii](https://zh.wikipedia.org/zh/ASCII)
- [ANSI_escape_code](https://en.wikipedia.org/wiki/ANSI_escape_code)
- [ANSI Escape Sequences](https://gist.github.com/fnky/458719343aabd01cfb17a3a4f7296797)
- [VT100 commands and control sequences](http://www.braun-home.net/michael/info/misc/VT100_commands.htm)
- [Go进阶19:如何开发多彩动感的终端UI应用](https://mojotv.cn/tutorial/golang-term-tty-pty-vt100)
- [TERMINAL TYPE DESCRIPTIONS SOURCE FILE](https://invisible-island.net/ncurses/terminfo.src.html)
- [控制终端代码 - Linux 控制终端转义和控制序列](http://manpages.ubuntu.com/manpages/bionic/zh_CN/man4/console_codes.4.html)
- [VT100 Terminal Package](https://github.com/xyproto/vt100/)