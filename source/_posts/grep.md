---
title: linux 基础的正则表达式 -- grep
date: 2016-5-8
---
正则表达式（Regular Expression）是通过一些特殊字符的排序，用以查找、替换、删除一行或多行文字字符串，简单的说，正则表达式就是用在字符串的处理上面的一项“表示试”。
**tips**
正则表达式并不是一个工具程序，而是一种字符串处理的标准依据，如果你想要以正则表达式的方式处理字符串，就得要使用支持正则表达式的工具程序才行，例如grep、sed、awk等。

###  grep
grep 是一个很常见也很常用的命令，它最重要的功能就是进行字符串数据的对比，然后将符合用户需求的字符串打印出来。需要说明的是grep在数据中查找一个字符串，是以整行为单位来进行数据的选取的。也就是说，假如一个文件内有10行，其中有两行具有你所查找的字符串，则将那两行显示在屏幕上，其他的就丢弃了。

-n和--color=auto 是grep常用的两个参数，如果每次使用的时候都要去敲这两个参数又太麻烦。可以在~/.bash_profile中假如这行：alias grep="grep --color -n"。

####  查找特定的字符串
1、从文件中查找good字符串，用如下方式：
```
[ggyy@gy-vm02 ~]$ grep good regular.txt 
1:"Open Source" is a good mechanism to develop programs.
9:Oh! The soup taste good.
```
加上-v参数则是取相反的结果，-i是不区分大小写

2、使用[]中括号来查找集合字符
如果要查找test和tast这两个单词时，可以使用如下表达式
```
[ggyy@gy-vm02 ~]$ grep 't[ea]st' regular.txt 
8:I can't finish the test.
9:Oh! The soup taste good.
```
[]中括号表示不论里面几个字符，都自会选则一个字符。

如果想要找到oo字符，但是前面没有g字符，可以使用如下表达式
```
[ggyy@gy-vm02 ~]$ grep '[^g]oo' regular.txt 
2:apple is my favorite food.
3:Football game is not use feet only.
18:google is the best tools for search keyword.
19:goooooogle yes!
```
如果oo前面不想有小写字符，可以使用如下表达式
```
[ggyy@gy-vm02 ~]$ grep '[^a-z]oo' regular.txt 
3:Football game is not use feet only.
```
当在一组集合字符中，如果该字符组是连续的，例如大/小写英文，数字等，就可以使用[a-z]，[A-Z]，[0-9]这样来表达，那么如果是要求字符串是数字和英文，就就它们写在一起,[a-zA-Z0-9]。
例如，我们要找出如有带数字的行数
```
[ggyy@gy-vm02 ~]$ grep [0-9] regular.txt 
5:However, this dress is about $ 3183 dollars.
15:You are the best is mean you are the no. 1.
```
3、行首与行尾字符^$
如果我们要查找以the字符串开头的行，使用如下表达式
```
[ggyy@gy-vm02 ~]$ grep '^the' regular.txt 
12:the symbol '*' is represented as start.
```
如果我们想要找开头是小写字符的那一行，可以这样
```
[ggyy@gy-vm02 ~]$ grep '^[a-z]' regular.txt 
2:apple is my favorite food.
4:this dress doesn't fit me.
10:motorcycle is cheap than car.
12:the symbol '*' is represented as start.
18:google is the best tools for search keyword.
19:goooooogle yes!
20:go! go! Let's go.
```
如我们不想开头是英文字符，可以使用如下表达式
```
[ggyy@gy-vm02 ~]$ grep '^[^a-zA-Z]' regular.txt 
1:"Open Source" is a good mechanism to develop programs.
21:# I am VBird
```
**tips**
^符号在字符集合[]之内表示反向选择，在[]之外，表示定位行首的意义，要区分清楚。

那如果我们想要找出以小数点 . 结尾的那一行，则这样
```
[ggyy@gy-vm02 ~]$ grep '\.$' regular.txt 
1:"Open Source" is a good mechanism to develop programs.
2:apple is my favorite food.
3:Football game is not use feet only.
4:this dress doesn't fit me.
10:motorcycle is cheap than car.
11:This window is clear.
12:the symbol '*' is represented as start.
15:You are the best is mean you are the no. 1.
16:The world <Happy> is the same with "glad".
17:I like dog.
18:google is the best tools for search keyword.
20:go! go! Let's go.
```
我们经常查看一些配置文件，里面有大量的注释信息和一些空行，那么我们可以使用如下方式
```
[ggyy@gy-vm02 ~]$ grep -v '^$' regular.txt | grep -v '^#' 
1:1:"Open Source" is a good mechanism to develop programs.
2:2:apple is my favorite food.
3:3:Football game is not use feet only.
4:4:this dress doesn't fit me.
5:5:However, this dress is about $ 3183 dollars.
6:6:GNU is free air not free beer.
7:7:Her hair is very beauty.
```
4、任意字符 . 和重复字符*

- .（小数点）：代表一定有一个任意字符的意思
-   *（星号）：代表重复前一个字符0到无穷多次的意思。
例如：
```
[ggyy@gy-vm02 ~]$ grep 'g..d' regular.txt 
1:"Open Source" is a good mechanism to develop programs.
9:Oh! The soup taste good.
16:The world <Happy> is the same with "glad".
```
上面的表达式是强调g与d之间一定要有两个字符，所以如果只有一个字符或者没有字符或者超过两个字符的都不会匹配。

再看如下例子：
```
[ggyy@gy-vm02 ~]$ grep 'ooo*' regular.txt 
1:"Open Source" is a good mechanism to develop programs.
2:apple is my favorite food.
3:Football game is not use feet only.
9:Oh! The soup taste good.
18:google is the best tools for search keyword.
19:goooooogle yes!
```
我们知道 * 表示重复前面一个字符0到无穷次个，因此，o* 代表具有空字符或一个以上的字符，特别注意，* 为0时表示空字符串，“[ggyy@gy-vm02 ~]$ grep  'o*' regular.txt”会把所有的数据都输出出来。
如果是“oo* ”，则第一个o肯定必须存在，第二个o则是可有可无的多个o，同理，当我们需要至少两个o以上的字符串时，就需要ooo* 

那如果我们想要字符串开头与结尾都是g，但是两个g之间仅能存在至少一个o，即gog、goog、gooog等
```
[ggyy@gy-vm02 ~]$ grep  'goo*g' regular.txt 
18:google is the best tools for search keyword.
19:goooooogle yes!
```
那如果我们想要找到g开头和g结尾的字符串，中间可有可无，如何表达了?
```
[ggyy@gy-vm02 ~]$ grep 'g.*g' regular.txt 
1:"Open Source" is a good mechanism to develop programs.
14:The gd software is a library for drafting programs.
18:google is the best tools for search keyword.
19:goooooogle yes!
20:go! go! Let's go.
```
5、限定连续的字符范围
我们在上面的例子中利用 . * 来设置0到无限个重复字符，那如何限定一个范围了。
比如说，我想找出2-4个o的连续字符串，这个时候，就得使用到限定范围的字符｛｝了。但是因为｛｝符号在shell里面是有特殊意义的，因此我们需要用上转义字符\来让他失去特殊意义才行。
```
[ggyy@gy-vm02 ~]$ grep 'o\{2,4\}' regular.txt 
1:"Open Source" is a good mechanism to develop programs.
2:apple is my favorite food.
3:Football game is not use feet only.
9:Oh! The soup taste good.
18:google is the best tools for search keyword.
19:goooooogle yes!
```
如果我们想要找到g后面有2个o以上，并且是g结尾的字符
```
[ggyy@gy-vm02 ~]$ grep 'go\{2,\}g' regular.txt 
18:google is the best tools for search keyword.
19:goooooogle yes!
```
这样就好了。

最后强调一下，正则表达式的特殊字符与一般在命令行的“通配符”并不相同，例如，在通配符中的* 代表的是零到无限多个字符的意思，但是在正则表达式中，则是重复0到无穷个前一个匹配字符的意思。
