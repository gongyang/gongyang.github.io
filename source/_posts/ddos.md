---
title : Linux防DDOS攻击软件--DDoS-Deflate
date: 2016-4-3
---
### 简介
[DDoS-Deflate](https://github.com/jgmdev/ddos-deflate)是一款免费的用来防御和减轻DDoS攻击的脚本。它通过netstat监测跟踪创建大量网络连接的IP地址，在检测到某个结点超过预设的限制时，该程序会通过APF或IPTABLES禁止或阻挡这些IP.
这个工具对大量模拟的假IP地址效果不是很明显

如何确认是否受到DDOS攻击？ 
使用下面这条命令进行查看
netstat -ntu | awk '{print $5}' | cut -d: -f1 | sort | uniq -c | sort -n
```
[bhuser@boohee-rc ~]$ netstat -ntu | awk '{print $5}' | cut -d: -f1 | sort | uniq -c | sort -n
      1 101.200.31.129
      1 101.200.31.147
      1 103.240.124.9
      1 192.168.1.18
      1 64.125.239.126
      1 Address
      1 servers)
      2 114.114.114.114
      2 192.168.1.64
      2 210.22.84.3
      2 58.246.136.4
      8 192.168.1.51
     16 192.168.1.16
     63 
     91 192.168.1.20
    107 192.168.1.44
    243 127.0.0.1
```
每个IP(内网IP除外)几个、十几个或几十个连接数都还算比较正常，如果某个IP连接数有几百上千就是攻击了。

### 安装
使用root用户
```
wget https://github.com/jgmdev/ddos-deflate/archive/master.zip
unzip master.zip
cd ddos-deflate-master
./install.sh
```
在执行install.sh可能会提示如下信息
```
[root@gy-vm03 ddos-deflate-master]# ./install.sh 
which: no tcpkill in (/usr/lib/jvm/java-7-oracle/bin:/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin:/root/bin)
error: Required dependency 'tcpkill' is missing.
```
***tcpkill***
当遇到TCP链接迟迟不能释放的情况，类似FIN_WAIT1、FIN_WAIT2的状态，释放时间不确定，而且对应的程序已经关闭，相应的端口也不再监听，无法通过杀进程来解决，这种情况下，为了快速恢复正常，不得不采用重启服务器的方法加以解决，Linux下可以借助dsniff包中含有tcpkill命令，该命令可以将上述状态的TCP链接加以清除

***下载dsniff rpm包***
wget http://mirrors.zju.edu.cn/epel/6/x86_64/libnet-1.1.6-7.el6.x86_64.rpm
wget http://mirrors.zju.edu.cn/epel/6/x86_64/libnids-1.24-1.el6.x86_64.rpm
wget http://mirrors.zju.edu.cn/epel/6/x86_64/dsniff-2.4-0.17.b1.el6.x86_64.rpm
其中libnet和libnids是两个依赖包

再次执行install.sh
```
[root@gy-vm03 ddos-deflate-master]# ./install.sh 

Installing DOS-Deflate 0.8

Adding: /etc/ddos/ddos.conf... (done)
Adding: /etc/ddos/ignore.ip.list... (done)
Adding: /etc/ddos/ignore.host.list... (done)
Adding: /usr/local/ddos/LICENSE... (done)
Adding: /usr/local/ddos/ddos.sh... (done)
Creating ddos script: /usr/local/sbin/ddos... (done)
Adding man page... (done)
Adding logrotate configuration... (done)

Setting up init script... (done)
Activating ddos service... (done)

Installation has completed!
Config files are located at /etc/ddos/

Please send in your comments and/or suggestions to:
https://github.com/jgmdev/ddos-deflate/issues
```
可以看到，主要配置文件在/etc/ddos/
```
[root@gy-vm03 ddos]# cd /etc/ddos/
[root@gy-vm03 ddos]# ls
ddos.conf  ignore.host.list  ignore.ip.list
[root@gy-vm03 ddos]#
```
ddos.conf是主配置文件
ignore.host.list 你可以添加主机名到这个文件里面做为白名单

```
boohee.com
www.boohee.com
```

ignore.ip.list  你可以添加IP到这个文件里面做为白名单

```
192.168.1.20
```

安装完成之后，系统默认启动了ddos
```
[root@gy-vm03 ddos]# ps aux | grep ddos
root      2616  0.1  0.1  11472  1456 pts/0    S    20:34   0:01 /bin/bash /usr/local/ddos/ddos.sh -l
root      5945  0.0  0.0 103252   836 pts/0    S+   20:48   0:00 grep ddos
[root@gy-vm03 ddos]# 
```

### 配置
ddos.conf是它的主配置文件
```
# Paths of the script and other files
PROGDIR="/usr/local/ddos" 
SBINDIR="/usr/local/sbin"
PROG="$PROGDIR/ddos.sh"
IGNORE_IP_LIST="ignore.ip.list"
IGNORE_HOST_LIST="ignore.host.list"
CRON="/etc/cron.d/ddos"
# Make sure your APF version is atleast 0.96
APF="/usr/sbin/apf"
CSF="/usr/sbin/csf"
IPT="/sbin/iptables"

# frequency in minutes for running the script as a cron job
# Caution: Every time this setting is changed, run the script with --cron
#          option so that the new frequency takes effect
FREQ=1  //检查时间间隔，默认1分钟

# frequency in seconds when running as a daemon
DAEMON_FREQ=5

# How many connections define a bad IP? Indicate that below.
NO_OF_CONNECTIONS=150 //最大连接数，超过这个数IP就会被屏蔽，默认150

# The firewall to use for blocking/unblocking, valid values are:
# auto, apf, csf and iptables
FIREWALL="iptables"

# An email is sent to the following address when an IP is banned.
# Blank would suppress sending of mails
EMAIL_TO="root" //当IP被屏蔽时给指定邮箱发送邮件，推荐使用，换成自己的邮箱即可

# Number of seconds the banned ip should remain in blacklist.
BAN_PERIOD=600 //禁用IP留在黑名单中的时间，默认600秒，可根据情况调整

# Connection states to block. See: man netstat
CONN_STATES="ESTABLISHED|SYN_SENT|SYN_RECV|FIN_WAIT1|FIN_WAIT2|TIME_WAIT|CLOSE_WAIT|LAST_ACK|CLOSING"
```
这个软件的关键程序就是这个ddos.sh脚本，原理是利用cron定时执行一次ddos.sh脚本，而ddos.sh通过执行netstat命令获取到异常的IP列表，添加到iptables中去。

**tips：**
**crontab任务是要执行ddos -c来创建的，更多参数请执行ddos --help查看**

### 测试
这里我们使用ab测试工具来检验一下
测试用到下面两台虚拟机
192.168.1.105  vm02 安装了ddos-deflate，并且是nginx服务器
192.168.1.104  vm03 测试机器
首先在vm03上面正常访问
```
[root@gy-vm03 ~]# curl -I ban1.gy.com
HTTP/1.1 200 OK
Server: Tengine/2.1.0
Date: Sun, 03 Apr 2016 13:13:53 GMT
Content-Type: text/html
Content-Length: 5
Last-Modified: Mon, 30 Nov 2015 17:18:12 GMT
Connection: keep-alive
ETag: "565c84d4-5"
Accept-Ranges: bytes
```
接下来在vm03上面使用ab测试工具
```
[root@gy-vm03 ~]# ab -c 1 -n 100 http://ban1.gy.com/inex.html
```
然后再到vm02上面查看
```
[root@gy-vm02 ~]# netstat -ntu | awk '{print $5}' | cut -d: -f1 | sort | uniq -c | sort -n
      1 192.168.168.1
      1 Address
      1 servers)
    100 192.168.10.104
```
使用iptable查看
```
[root@gy-vm02 ~]# iptables -L -n
Chain INPUT (policy ACCEPT)
target     prot opt source               destination         
ACCEPT     all  --  0.0.0.0/0            0.0.0.0/0           state RELATED,ESTABLISHED 
ACCEPT     icmp --  0.0.0.0/0            0.0.0.0/0           
ACCEPT     all  --  0.0.0.0/0            0.0.0.0/0           
ACCEPT     all  --  0.0.0.0/0            0.0.0.0/0           
ACCEPT     tcp  --  0.0.0.0/0            0.0.0.0/0           state NEW tcp dpt:22 
ACCEPT     tcp  --  0.0.0.0/0            0.0.0.0/0           state NEW tcp dpt:873 
ACCEPT     tcp  --  0.0.0.0/0            0.0.0.0/0           state NEW tcp dpt:80 
REJECT     all  --  0.0.0.0/0            0.0.0.0/0           reject-with icmp-host-prohibited 

Chain FORWARD (policy ACCEPT)
target     prot opt source               destination         
DOCKER     all  --  0.0.0.0/0            0.0.0.0/0           
ACCEPT     all  --  0.0.0.0/0            0.0.0.0/0           ctstate RELATED,ESTABLISHED 
ACCEPT     all  --  0.0.0.0/0            0.0.0.0/0           
REJECT     all  --  0.0.0.0/0            0.0.0.0/0           reject-with icmp-host-prohibited 
ACCEPT     all  --  0.0.0.0/0            0.0.0.0/0           

Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination         

Chain DOCKER (1 references)
target     prot opt source               destination
```
此时并没有添加进来

再次到vm03上面测试
```
[root@gy-vm03 ~]# ab -c 1 -n 300 http://ban1.gy.com/inex.html
```

然后到vm02上面进行查看
```
[root@gy-vm02 ~]# iptables -L -n
Chain INPUT (policy ACCEPT)
target     prot opt source               destination         
DROP       all  --  192.168.10.104       0.0.0.0/0           
ACCEPT     all  --  0.0.0.0/0            0.0.0.0/0           state RELATED,ESTABLISHED 
ACCEPT     icmp --  0.0.0.0/0            0.0.0.0/0           
ACCEPT     all  --  0.0.0.0/0            0.0.0.0/0           
ACCEPT     all  --  0.0.0.0/0            0.0.0.0/0           
ACCEPT     tcp  --  0.0.0.0/0            0.0.0.0/0           state NEW tcp dpt:22 
ACCEPT     tcp  --  0.0.0.0/0            0.0.0.0/0           state NEW tcp dpt:873 
ACCEPT     tcp  --  0.0.0.0/0            0.0.0.0/0           state NEW tcp dpt:80 
REJECT     all  --  0.0.0.0/0            0.0.0.0/0           reject-with icmp-host-prohibited 

Chain FORWARD (policy ACCEPT)
target     prot opt source               destination         
DOCKER     all  --  0.0.0.0/0            0.0.0.0/0           
ACCEPT     all  --  0.0.0.0/0            0.0.0.0/0           ctstate RELATED,ESTABLISHED 
ACCEPT     all  --  0.0.0.0/0            0.0.0.0/0           
REJECT     all  --  0.0.0.0/0            0.0.0.0/0           reject-with icmp-host-prohibited 
ACCEPT     all  --  0.0.0.0/0            0.0.0.0/0           

Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination         

Chain DOCKER (1 references)
target     prot opt source               destination 
```
此时192.168.10.104被添加了进来
并且在bhvm02上面查看日志
```
[root@gy-vm02 ~]# tail /var/log/ddos.log 
[2016-03-29 20:02:28] daemon started
[2016-03-29 20:02:50] banned 192.168.1.124 with 13557 connections for ban period 600
[2016-03-29 20:12:55] unbanned 192.168.1.124
[2016-03-29 20:13:00] banned 192.168.1.124 with 600 connections for ban period 600
[2016-03-29 20:23:17] unbanned 192.168.1.124
[2016-03-29 23:31:56] daemon started
[2016-03-30 09:50:37] daemon started
[2016-04-03 21:12:23] added cron job
[2016-04-03 21:22:06] daemon started
[2016-04-03 21:22:06] banned 192.168.10.104 with 900 connections for ban period 200
[root@gy-vm02 ~]#
```

### 卸载
```
cd ddos-deflate-master
./uninstall.sh
```
