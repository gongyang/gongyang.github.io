---
title : linux实用工具之--AWK
date: 2016-6-26
---
AWK是由 Alfred Aho、Peter Weinberger、Brian Kernighan三人创造出来的一款非常棒的数据处理工具。
相比于sed作用于一整行的数据处理方式，awk则是将一行数据分成数个“字段”来处理，而默认的字段分隔符为空格键或[tab]键。

**使用语法：**
> awk '条件类型1{动作1} 条件类型2{动作2}...' filename

**实例 1**
使用last查看最后10条登录记录，并只查看登录用户名和IP
```
[ggyy@gy-vm02 ~]$ last -n10 |awk '{print $1 "\t" $3}'
ggyy	192.168.168.1
reboot	boot
ggyy	192.168.168.1
ggyy	192.168.168.1
reboot	boot
ggyy	192.168.168.1
reboot	boot
ggyy	192.168.168.1
ggyy	192.168.168.1
reboot	boot
```
print是awk最常使用的动作，通过print的功能将字段数据列出来。其中的```$1..$n```表示第几例，```$0```表示整个行。“\t” 的意思是```$1``` 和 ```$3``` 之间用[tab]隔开。

awk的处理数据的流程如下；
1、读入第一行，并将第一行的数据填入 ```$0,$1,$2``` 等变量中；
2、依据条件类型的限制，判断是否需要进行后面的动作；
3、做完所有的动作与条件类型；
4、若还有后续的数据，则重复上面的1-3的步骤，直到所有的数据都读完为止。

通过上面的讲叙，我们知道awk是以行为一次的处理单位，以字段为最小的处理单位。那么awk怎么知道到底这个数据有几行几列？我们来了解一下awk内置的变量
> NF：每一行（$0）拥有的字段总数
> NR：目前awk所处理的是“第几行”数据
> FS：目前的分隔字符，默认是空格键

**tips：**
由于单引号是awk的固定用法，awk后续的所有动作都是以单引号括住的，所以使用print打印时，非变量的文字部分，包括 \t ，都需要使用双引号括起来。

#### AWK的逻辑运算符
逻辑运算符是作用在条件里面，awk主要有的逻辑运算符有：
```
 > 大于
 < 小于
 >= 大于等于
 <= 小于等于
 == 等于
 != 不等于
```
**实例2**
查看/etc/password里面UID小于5，且只显示用户名和UID
```
[ggyy@gy-vm02 ~]$ awk '{FS=":"}$3<5{print $1 "\t" $3}' /etc/passwd
root:x:0:0:root:/root:/bin/bash	
bin	1
daemon	2
adm	3
lp	4
[ggyy@gy-vm02 ~]$
```
我们可以看到第一行数据并没有按照我们的想法输出，这是因为我们定义的```FS=“:”``` 动作要从第二行才开始生效，第一行还是按照原来的方式输出了，所以这个时候需要加上BEGIN关键字：
```
[ggyy@gy-vm02 ~]$ awk 'BEGIN{FS=":"}$3<5{print $1 "\t" $3}' /etc/passwd
root	0
bin	1
daemon	2
adm	3
lp	4
```

**实例3**
awk还可以像grep去匹配数据
```
[ggyy@gy-vm02 ~]$ awk '$6 ~ /FIN|TIME/ {print NR,$4,$5,$6}' 1.txt 
5 coolshell.cn:80 124.205.5.146:18245 TIME_WAIT
6 coolshell.cn:80 61.140.101.185:37538 FIN_WAIT2
9 coolshell.cn:80 116.234.127.77:11502 FIN_WAIT2
11 coolshell.cn:80 183.60.215.36:36970 TIME_WAIT
13 coolshell.cn:80 124.152.181.209:26825 FIN_WAIT1
```
还可以取反
```
[ggyy@gy-vm02 ~]$ awk '$6 !~ /FIN|TIME/ {print NR,$4,$5,$6}' 1.txt 
1 Local-Address Foreign-Address State
2 0.0.0.0:3306 0.0.0.0:* LISTEN
3 0.0.0.0:80 0.0.0.0:* LISTEN
4 127.0.0.1:9000 0.0.0.0:* LISTEN
7 coolshell.cn:80 110.194.134.189:103 ESTABLISHED
8 coolshell.cn:80 123.169.124.111:49809 ESTABLISHED
10 coolshell.cn:80 123.169.124.111:49829 ESTABLISHED
12 coolshell.cn:80 61.148.242.38:30901 ESTABLISHED
14 :::22 :::* LISTEN
```

除此之外，AWK还有很多高级语法，这里就不详细讲叙了，了解基础就可以应付大部分的操作了。
