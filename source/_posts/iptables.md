---
title : Linux 的封包过滤软件--iptables
date: 2016-4-17
---
### 前言
iptables，对于接触过linux的人来说，对这个概念不会感到陌生。它是linux系统自带的一款软件防火墙。以前我一直对这个知识点掌握的比较零散，今天决定系统的来学习一下。早期的linux并不是使用iptables，在内核版本2.0是使用的 ipfwadm 这个防火墙机制，在内核版本2.2是ipchains 这个防火墙机制，在内核版本2.6之后就是使用的iptables了。

了解了它的发展史，接下来进入正题。

### 工作机制
iptables 是利用封包过滤的机制， 所以他会分析封包的表头数据。根据表头数据与定义的规则来决定该封包是否可以进入主机或者是被丢弃。 意思就是说：根据封包的分析资料 "比对" 你预先定义的规则内容， 若封包数据与规则内容相同则进行动作，否则就继续下一条规则的比对！
所以，重点就在比对与分析顺序上。

### iptables的表格 (table) 与链 (chain)
Linux 的 iptables 至少就有三个表格，包括管理本机进出的 filter 、管理后端主机 (防火墙内部的其他计算机) 的 nat 、管理特殊旗标使用的 mangle (较少使用) ，表格和链的用途分别是这样的

**filter (过滤器)**
主要跟进入 Linux 本机的封包有关，这个是预设的 table

- INPUT：主要与想要进入我们 Linux 本机的封包有关
- OUTPUT：主要与我们 Linux 本机所要送出的封包有关
- FORWARD：这个咚咚与 Linux 本机比较没有关系， 他可以转递封包到后端的计算机中，与下列 nat table 相关性较高

**nat (地址转换)**
是 Network Address Translation 的缩写， 这个表格主要在进行来源与目的之 IP 或 port 的转换，与 Linux 本机较无关，主要与 Linux 主机后的局域网络内计算机较有相关

- PREROUTING：在进行路由判断之前所要进行的规则(DNAT/REDIRECT)
- POSTROUTING：在进行路由判断之后所要进行的规则(SNAT/MASQUERADE)
- OUTPUT：与发送出去的封包有关

**mangle (破坏者)**
这个表格主要是与特殊的封包的路由旗标有关， 早期仅有 PREROUTING 及 OUTPUT 链，不过从 kernel 2.4.18 之后加入了 INPUT 及 FORWARD 链。 由于这个表格与特殊旗标相关性较高，所以像咱们这种单纯的环境当中，较少使用 mangle 这个表格

### iptables 语法
了解了概念性的知识，我来进入实战。我们在安装好linux系统之后，系统会自动启动一个防火墙规则，不过这个可能不是我们想要的模式，因此我们需要额外进行一些修订的行为。不过，我们在进行练习的过程中，最好不要在远程的机器上面进行操作，因为 iptables 的指令会将网络封包进行过滤及抵挡的动作，你很有可能一不小心将自己关在家门外。

####  查看与清除iptables规则
**查看语法**
iptables [-t tables] [-L] [-nv]
选项与参数：
-t ：后面接 table ，例如 nat 或 filter ，若省略此项目，则使用默认的 filter
-L ：列出目前的 table 的规则
-n ：不进行 IP 与 HOSTNAME 的反查，显示讯息的速度会快很多！
-v ：列出更多的信息，包括通过该规则的封包总位数、相关的网络接口等
例如查看本机的防火墙规则：
```
[root@gy-vm04 ~]# iptables -L -n
Chain INPUT (policy ACCEPT) //针对 INPUT 链，且预设政策为可接受
target     prot opt source  //说明栏              destination         
ACCEPT     all  --  0.0.0.0/0            0.0.0.0/0           state RELATED,ESTABLISHED  //第 1 条规则
ACCEPT     icmp --  0.0.0.0/0            0.0.0.0/0 //第 2 条规则        
ACCEPT     all  --  0.0.0.0/0            0.0.0.0/0 //第 3 条规则  以下类推   
ACCEPT     tcp  --  0.0.0.0/0            0.0.0.0/0           state NEW tcp dpt:22  
ACCEPT     tcp  --  0.0.0.0/0            0.0.0.0/0           state NEW tcp dpt:80  
ACCEPT     tcp  --  0.0.0.0/0            0.0.0.0/0           state NEW tcp dpt:443 
REJECT     all  --  0.0.0.0/0            0.0.0.0/0           reject-with icmp-host-prohibited 

Chain FORWARD (policy ACCEPT) //针对 FORWARD 链，且预设政策为可接受
target     prot opt source               destination         
DOCKER     all  --  0.0.0.0/0            0.0.0.0/0           
ACCEPT     all  --  0.0.0.0/0            0.0.0.0/0           ctstate RELATED,ESTABLISHED 
ACCEPT     all  --  0.0.0.0/0            0.0.0.0/0           
REJECT     all  --  0.0.0.0/0            0.0.0.0/0           reject-with icmp-host-prohibited 
ACCEPT     all  --  0.0.0.0/0            0.0.0.0/0           

Chain OUTPUT (policy ACCEPT) //针对 OUTPUT 链，且预设政策为可接受
target     prot opt source               destination         

Chain DOCKER (1 references) //这是关于docker的自定义链
target     prot opt source               destination         
[root@gy-vm04 ~]#
```
如果要查看关于nat 的table规则
```
[root@gy-vm04 ~]# iptables -t nat -L -n
Chain PREROUTING (policy ACCEPT)
target     prot opt source               destination         
DOCKER     all  --  0.0.0.0/0            0.0.0.0/0           ADDRTYPE match dst-type LOCAL 

Chain POSTROUTING (policy ACCEPT)
target     prot opt source               destination         
MASQUERADE  all  --  172.17.0.0/16        0.0.0.0/0           

Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination         
DOCKER     all  --  0.0.0.0/0           !127.0.0.0/8         ADDRTYPE match dst-type LOCAL 

Chain DOCKER (2 references)
target     prot opt source               destination
```
上面的每一个Chain就是所谓的链，Chain 那一行里面括号的 policy 就是预设的政策，什么是policy，就是当一个封包不匹配所有的规则，就默认执行这个政策。其他的参数说明

- target：代表进行的动作， ACCEPT 是放行，而 - REJECT 则是拒绝，此外，尚有 DROP (丢弃) 的项目！
- prot：代表使用的封包协议，主要有 tcp, udp 及 icmp 三种封包格式；
- opt：额外的选项说明
- source ：代表此规则是针对哪个来源 IP进行限制？
- destination ：代表此规则是针对哪个目标 IP进行限制？

**清除规则**
如何删除默认的规则，来建立我们自己想要定义的规则了。
语法：
iptables [-t tables] [-FXZ]
选项与参数：
-F ：清除所有的已订定的规则；
-X ：杀掉所有使用者 "自定义" 的 chain (应该说的是 tables ）啰；
-Z ：将所有的 chain 的计数与流量统计都归零

***切记***
这个操作一定要小心，这三个指令会将本机防火墙的所有规则都清除，但却不会改变预设政策 (policy) ， 所以如果你不是在本机下达这三行指令时，很可能你会被自己挡在家门外

#### 定义预设policy（政策）
语法：
 iptables [-t nat] -P [INPUT,OUTPUT,FORWARD] [ACCEPT,DROP]
 选项与参数：
-P ：定义政策( Policy )。注意，这个 P 为大写啊！
ACCEPT ：该封包可接受
DROP   ：该封包直接丢弃，不会让 client 端知道为何被丢弃。
例如将本机的INPUT 设定为 DROP ，其他设定为 ACCEPT
```
[root@gy-vm04 ~]# iptables -P INPUT DROP
[root@gy-vm04 ~]# iptables -P OUTPUT  ACCEPT
[root@gy-vm04 ~]# iptables -P FORWARD ACCEPT
```
**tips**
我们在修改完规则之后，记得使用iptables-save这条命名保存，不然的话，我们以后在重启iptables的服务之后，规则就不存在了。

####  针对IP已经网口的设置
语法：
 iptables [-AI 链名] [-io 网络接口] [-p 协议]  [-s 来源IP/网域] [-d 目标IP/网域] -j [ACCEPT|DROP|REJECT|LOG]

选项与参数：
-AI 链名：针对某的链进行规则的 "插入" 或 "累加"
    -A ：新增加一条规则，该规则增加在原本规则的最后面。例如原本已经有四条规则，
         使用 -A 就可以加上第五条规则！
    -I ：插入一条规则。如果没有指定此规则的顺序，默认是插入变成第一条规则。
         例如原本有四条规则，使用 -I 则该规则变成第一条，而原本四条变成 2~5 号
    链 ：有 INPUT, OUTPUT, FORWARD 等，此链名称又与 -io 有关，请看底下。

-io 网络接口：设定封包进出的接口规范
    -i ：封包所进入的那个网络接口，例如 eth0, lo 等接口。需与 INPUT 链配合；
    -o ：封包所传出的那个网络接口，需与 OUTPUT 链配合；

-p 协定：设定此规则适用于哪种封包格式
   主要的封包格式有： tcp, udp, icmp 及 all 。

-s 来源 IP/网域：设定此规则之封包的来源项目，可指定单纯的 IP 或包括网域，例如：
   IP  ：192.168.0.100
   网域：192.168.0.0/24, 192.168.0.0/255.255.255.0 均可。
   若规范为『不许』时，则加上 ! 即可，例如：
   -s ! 192.168.100.0/24 表示不许 192.168.100.0/24 之封包来源；

-d 目标 IP/网域：同 -s ，只不过这里指的是目标的 IP 或网域。

-j ：后面接动作，主要的动作有接受(ACCEPT)、丢弃(DROP)、拒绝(REJECT)及记录(LOG)

例如我们对于lo接口所有封包添加为接受
```
[root@gy-vm04 ~]# iptables -A INPUT -i lo -j ACCEPT
```
**tips**
如果没有没有列出 -s, -d 等等的规则，这表示：不论封包来自何处或去到哪里，只要是来自 lo 这个接口，就予以接受！

由于我比较喜欢使用Xshell登录来操作虚拟机，所以一般会把我本机的IP添加为ACCEPT
```
[root@gy-vm04 ~]# iptables -A INPUT -s 192.168.168.2 -j ACCEPT
```
这样我才能登录到这台虚拟机上面去。

#### 针对端口的设定
语法：
iptables [-AI 链] [-io 网络接口] [-p tcp,udp]  [-s 来源IP/网域] [--sport 端口范围] [-d 目标IP/网域] [--dport 端口范围] -j [ACCEPT|DROP|REJECT]
例如：
打开eth0上的8080端口
```
[root@gy-vm04 ~]# iptables -A INPUT -i eht0  --dport 8080 -j ACCEPT
iptables v1.4.7: unknown option `--dport'
Try `iptables -h' or 'iptables --help' for more information.
```
额，怎么报错了，提示不知道参数--dport。
这是因为仅有 tcp 与 udp 封包具有端口，因此你想要使用 --dport, --sport 时，得要加上 -p tcp 或 -p udp 的参数才会成功
正确的写法是：
```
[root@gy-vm04 ~]# iptables -A INPUT -i eht0 -p tcp --dport 8080 -j ACCEPT
```
**tips**
我们可以指定连续的端口
例如：--dport 1000:2000

####  iptables 外挂模块：mac 与 state
语法：
 iptables -A INPUT [-m state] [--state 状态]
 选项与参数：
-m ：一些 iptables 的外挂模块，主要常见的有：
     state ：状态模块
     mac   ：网络卡硬件地址 (hardware address)
--state ：一些封包的状态，主要有：
     INVALID    ：无效的封包，例如数据破损的封包状态
     ESTABLISHED：已经联机成功的联机状态；
     NEW        ：想要新建立联机的封包状态；
     RELATED    ：这个最常用！表示这个封包是与我们主机发送出去的封包有关

例如：
只要已建立或相关封包就予以通过，只要是不合法封包就丢弃
```
[root@gy-vm04 ~]# iptables -A INPUT -m state --state RELATED,ESTABLISHED -j ACCEPT
[root@gy-vm04 ~]# iptables -A INPUT -m state --state INVALID -j DROP
```
iptables还可以针对网卡来进行放行与防御
语法：
iptables -A INPUT -m mac --mac-source aa:bb:cc:dd:ee:ff -j ACCEPT
选项与参数：
--mac-source ：就是来源主机的 MAC

#### ICMP 封包规则
语法：
iptables -A INPUT [-p icmp] [--icmp-type 类型] -j ACCEPT
选项与参数：
--icmp-type ：后面必须要接 ICMP 的封包类型，也可以使用代号，例如 8  代表 echo request 的意思。
默认系统是对所有类型都ACCEPT
