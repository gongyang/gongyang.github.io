---
title : command中单双引的区别
date: 2016-5-15
---
在讲今天的主题之前，我们先来了解一下shell中一些基础知识。
当我们在shell prompt后面敲打键盘, 直到按下Enter键的时候，你输入的文字就是command line了， 然后shell才会以进程的方式执行你所交给它的命令。 但是，你又可知道：你在command line中输入的每一个文字， 对shell来说，是有类别之分的呢？

简单而言，command line的每一个charactor， 分为如下两种：

- literal：也就是普通的纯文字，对shell来说没特殊功能；
- meta: 对shell来说，具有特定功能的特殊保留元字符。

literal就是我们正常输入的字符串， 像 abcd、123456 这些 "文字" 都是 literal，而常用的meta有：

- IFS（ Interchange Format Syntax ）：有space或者tab或者Enter三者之一组成 (我们常用 space)
- CR（回车键）: 由Enter产生；

IFS是用来拆解command line中每一个词 (word) 用的， 因为shell command line是按词来处理的。 而CR则是用来结束command line用的，这也是为何我们敲Enter键， 命令就会运行的原因。
```
除了常用的IFS与CR, 常用的 meta 还有：
= ：  设定变量。
$ ：  做变量或运算替换(请不要与 shell prompt 搞混了)。
> ：  重定向 stdout（标准输出standard out）。
< ：  重定向 stdin（标准输入standard in）。
|：   管道命令。
& ：  重定向 file descriptor （文件描述符），或将命令置于后台执行。
( )： 將其內的命令置于 nested subshell （嵌套的子shell）执行，或用于运算或命令替换。
{ }： 將其內的命令置于 non-named function（未命名函数） 中执行，或用在变量替换的界定范围。
; ：  在前一个命令结束时，而忽略其返回值，继续执行下一個命令。
&& ： 在前一個命令结束时，若返回值为 true，继续执行下一個命令。
|| ： 在前一個命令结束时，若返回值为 false，继续执行下一個命令。
!：   执行 history 列表中的命令
```
假如我们需要在command line中将这些保留元字符的功能关闭的话， 就需要 quoting 处理了。
在bash中，常用的 quoting 有以下三种方法：

- hard quote：''(单引号)，凡在 hard quote 中的所有 meta 均被关闭；
- soft quote：""（双引号），在soft quote中的大部分meta都会被关闭，但某些保留（如$）。
- escape：\ （反斜线），只有紧接在escape（跳脱字符）之后的单一meta才被关闭。

下面我们通过实验来对quote进一步的了解
```
[ggyy@gy-vm02 ~]$ A=B C
-bash: C: command not found
[ggyy@gy-vm02 ~]$ echo $A

[ggyy@gy-vm02 ~]$ A="B C"
[ggyy@gy-vm02 ~]$ echo $A
B C
[ggyy@gy-vm02 ~]$
```
在给第一个A赋值的时候，由于空白符没有被关闭，command line 将被解释为： A=B 然后碰到<IFS>，接着执行C命令，所以会提示一个C命令没找到，并且A赋值也没有成功。
在第二次给 A 变量赋值时，由于空白符被置于 soft quote 中， 因此被关闭，不在作为IFS；
事实上，空白符无论在 soft quote 还是在 hard quote 中， 均被关闭。Enter 键字符亦然：
```
[ggyy@gy-vm02 ~]$ A='B C'
[ggyy@gy-vm02 ~]$ echo $A
B C
```
我们再来看一个例子：
```
[ggyy@gy-vm02 ~]$ a='1
> 2
> 3
> '
[ggyy@gy-vm02 ~]$ echo "$a"
1
2
3

[ggyy@gy-vm02 ~]$ echo $a
1 2 3
```
在上例中，由于 <enter> 被置于 hard quote 当中，因此不再作为 CR 字符來处理。
这里的 <enter> 单纯只是一个断行符号(new-line)而已，由于 command line 并沒得到 CR 字符，
因此进入第二個 shell prompt (PS2，以 > 符号表示)，command line 并不会结束，
直到第四行，我们输入的 <enter> 并不在  hard quote 里面，此时，command line 碰到 CR 字符，于是结束、交给 shell 來处理。
但是为什么``` echo "$a" ``` 和 ```echo $a``` 会输出不一样的结果了。
这是因为 ```echo $a ```时的变量沒置于 soft quote 中，因此当变量替换完成后并作命令行重组时，<enter> 会被解释为 IFS ，而不是解释为 New Line 字符。

其实我们可以用 escape 亦可关闭 CR 字符：
```
[ggyy@gy-vm02 ~]$ a=1\
> 2\
> 3
[ggyy@gy-vm02 ~]$ echo $a
123
```
上例中，第一个 <enter> 跟第二个 <enter> 均被 escape 字符关闭了，因此也不作为 CR 來处理，
但第三个 <enter> 由于没有被转义，因此作为 CR 结束 command line 。

至于 soft quote 跟 hard quote 的不同，主要是对于某些 meta 的关闭与否，以 $ 來作说明：
```
[ggyy@gy-vm02 ~]$ a=1
[ggyy@gy-vm02 ~]$ echo "$a"
1
[ggyy@gy-vm02 ~]$ echo '$a'
$a
```
在第一个echo命令行中，$ 被置于 soft quote 中，将不被关闭， 因此继续处理变量替换， 因此，echo将 a 的变量值输出到屏幕，也就是 1 的结果。

在第二个echo命令行中，```$ ```被置于 hard quote 中，则被关闭， 因此，```$``` 只是一个 ```$``` 符号，并不会用来做变量替换处理， 因此结果是``` $``` 符号后面接一个 a 字母：```$a```。

下面这个例子就能完美反映出他们的不同
```
[ggyy@gy-vm02 ~]$ a=b\ c
[ggyy@gy-vm02 ~]$ echo "$a"
b c
[ggyy@gy-vm02 ~]$ echo $a
b c
[ggyy@gy-vm02 ~]$ echo '"$a"'
"$a"
[ggyy@gy-vm02 ~]$ echo "'$a'"
'b c'
```

