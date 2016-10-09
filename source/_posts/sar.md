---
title : linux 系统运维工具--sar
date: 2016-7-28
---
## linux 系统运维工具--sar
sar（System Activity Reporter系统活动情况报告）是目前 Linux 上最为全面的系统性能分析工具之一，可以从多方面对系统的活动进行报告，包括：文件的读写情况、系统调用的使用情况、磁盘I/O、CPU效率、内存使用状况等

### 软件安装包
sar是由sysstat软件包提供，sysstat除了提供sar命令之外，还提供了下面命令
```
/usr/bin/cifsiostat #统计CIFS协议的网络文件系统的 I/O状态数据。
/usr/bin/iostat #输出CPU的统计信息和所有I/O设备的输入输出（I/O）统计信息。
/usr/bin/mpstat #关于CPU的详细信息（单独输出或者分组输出）。
/usr/bin/pidstat #关于运行中的进程/任务、CPU、内存等的统计信息。
/usr/bin/sadf #用于以不同的数据格式（CVS或者XML）来格式化sar工具的输出。
/usr/bin/sar #保存并输出不同系统资源（CPU、内存、IO、网络、内核等。。。）的详细信息。
```
除此之外，安装sysstat包后，默认创建一个/etc/cron.d/sysstat文件，其默认内容为：
```
# Run system activity accounting tool every 10 minutes
*/10 * * * * root /usr/lib64/sa/sa1 1 1
# 0 * * * * root /usr/lib64/sa/sa1 600 6 &
# Generate a daily summary of process accounting at 23:53
53 23 * * * root /usr/lib64/sa/sa2 -A
```
sa1、sa2是两个shell脚本

- sa1：调用sadc（二进制文件），将数据收集到二进制日志文件的一个Shell脚本。sa1命令还确保每天使用不同的文件。每隔十分钟运行一次该命令，最好不要改这个值，这是对一般系统折中的值。二进制日志文件存放在/var/log/sa/目录下，命名为sa${DATE}，这个文件是一个二进制的文件，需要使用sar -f 接文件名来查看。sa1用作 sadc 的前端程序。
- sa2：是将当日二进制日志文件中所有的数据转储到文本文件的另一个Shell脚本，参数-A是表示将所有sar的信息都写到文本并保存放在/var/log/sa/目录下，命名为sar${DATE}，这是一个纯文本文件，可以直接查看。sa2用作 sar 的前端程序。
- sadc ：系统动态数据收集工具，收集的数据被写入一个二进制的文件中，它被用作 sar 工具的后端

### sar命令格式
sar [ -A ] [ -B ] [ -b ] [ -C ] [ -D ] [ -d ] [ -F [ MOUNT ] ] [ -H ] [ -h ] [ -p ] [ -q ] [ -R ] [ -r [ ALL ] ] [ -S ] [ -t ] [ -u [ ALL ] ] [ -V ] [ -v ] [ -W ] [ -w ] [ -y ] [ --sadc ] [ -I { int [,...] | SUM | ALL | XALL } ] [ -P { cpu [,...] | ALL } ] [ -m { keyword [,...] | ALL } ] [ -n { keyword [,...] | ALL } ] [ -j { ID | LABEL | PATH | UUID | ... } ] [ -f [ filename ] | -o [ filename ] | -[0-9]+ ] [ -i interval ] [ -s [ hh:mm[:ss] ] ] [ -e [ hh:mm[:ss] ] ] [ interval [ count ] ]

- interval : 为取样时间间隔
- count : 为输出次数，若省略此项，会一直输出下去
```
-A：所有报告的总和
-u：输出CPU使用情况的统计信息
-v：输出inode、文件和其他内核表的统计信息
-d：输出每一个块设备的活动信息
-r：输出内存和交换空间的统计信息
-b：显示I/O和传送速率的统计信息
-a：文件读写情况
-c：输出进程统计信息，每秒创建的进程数
-R：输出内存页面的统计信息
-y：终端设备活动情况
-w：输出系统交换活动信息
-n：{DEV|EDEV|NFS|NFSD|SOCK|ALL}	分析输出网络设备状态统计信息。
DEV	报告网络设备的统计信息
EDEV	报告网络设备的错误统计信息
NFS	报告 NFS 客户端的活动统计信息
NFSD	报告 NFS 服务器的活动统计信息
SOCK	报告网络套接字（sockets）的使用统计信息
ALL	报告所有类型的网络活动统计信息
-o：filename	将输出信息保存到文件 filename
-f：filename	从文件 filename 读取数据信息。filename 是使用-o 选项时生成的文件。
-s：hh:mm:ss	指定输出统计数据的起始时间
-e：hh:mm:ss	指定输出统计数据的截至时间，默认为18:00:00
```

### sar 常用举例
**CPU资源监控**
每3秒采集一次数据，总共采集10次，并将数据以二进制方式保存到文件
```
[bhuser@bhqa ~]$ sar -u -o cpuinfo 3 10
Linux 2.6.32-504.8.1.el6.x86_64 (bhqa) 	2016年07月28日 	_x86_64_	(2 CPU)

00时35分50秒     CPU     %user     %nice   %system   %iowait    %steal     %idle
00时35分53秒     all      8.70      0.00      3.24      1.54      0.00     86.52
00时35分56秒     all      8.05      0.00      3.08      1.54      0.00     87.33
00时35分59秒     all      8.58      0.00      3.26      1.37      0.00     86.79
00时36分02秒     all      8.56      0.00      3.25      1.03      0.00     87.16
00时36分05秒     all      8.61      0.00      3.79      3.44      0.00     84.17
00时36分08秒     all      8.92      0.00      3.77      2.40      0.00     84.91
00时36分11秒     all      9.23      0.00      3.25      0.85      0.00     86.67
00时36分14秒     all      8.76      0.00      3.26      1.20      0.00     86.77
00时36分17秒     all      9.56      0.00      3.75      0.51      0.00     86.18
00时36分20秒     all      8.25      0.00      4.30      1.20      0.00     86.25
平均时间:     all      8.72      0.00      3.50      1.51      0.00     86.27
```
输出项说明：

CPU：all 表示统计信息为所有 CPU 的平均值。
%user：显示在用户级别(application)运行使用 CPU 总时间的百分比。
%nice：CPU用来执行有用户级别优先级别的任务的时间的百分比。
%system：在核心级别(kernel)运行所使用 CPU 总时间的百分比。
%iowait：显示用于等待I/O操作占用 CPU 总时间的百分比。
%steal：管理程序(hypervisor)为另一个虚拟进程提供服务而等待虚拟 CPU 的百分比。
%idle：显示 CPU 空闲时间占用 CPU 总时间的百分比。

1. 若 %iowait 的值过高，表示硬盘存在I/O瓶颈
2. 若 %idle 的值高但系统响应慢时，有可能是 CPU 等待分配内存，此时应加大内存容量
3. 若 %idle 的值持续低于10，则系统的 CPU 处理能力相对较低，表明系统中最需要解决的资源是 CPU 。
如果要查看二进制文件cpuinfo中的内容，需键入如下sar命令：
sar  -f cpuinfo

**I/O信息监控**
```
[bhuser@bhqa ~]$ sar -b 2 5
Linux 2.6.32-504.8.1.el6.x86_64 (bhqa) 	2016年07月28日 	_x86_64_	(2 CPU)

00时56分53秒       tps      rtps      wtps   bread/s   bwrtn/s
00时56分55秒     37.11      4.64     32.47     65.98    379.38
00时56分57秒    540.51      0.00    540.51      0.00   8459.49
00时56分59秒      2.05      0.00      2.05      0.00     16.41
00时57分01秒      4.08      4.08      0.00    277.55      0.00
00时57分03秒      1.03      0.00      1.03      0.00      8.25
平均时间:    117.04      1.75    115.30     68.99   1774.13
```
tps	每秒钟物理设备的 I/O 传输总量
rtps	每秒钟从物理设备请求读次数
wtps	每秒钟向物理设备请求写入次数
bread/s	每秒钟从物理设备 读入的数据量，单位为 块/s
bwrtn/s每秒钟向物理设备写入的数据量，单位为 块/s

**进程队列信息监控**
```
[bhuser@bhqa ~]$ sar -q 3 5
Linux 2.6.32-504.8.1.el6.x86_64 (bhqa) 	2016年07月28日 	_x86_64_	(2 CPU)

00时58分52秒   runq-sz  plist-sz   ldavg-1   ldavg-5  ldavg-15
00时58分55秒         3      1077      0.00      0.00      0.00
00时58分58秒         1      1077      0.00      0.00      0.00
00时59分01秒         0      1077      0.00      0.00      0.00
00时59分04秒         0      1077      0.00      0.00      0.00
00时59分07秒         0      1077      0.00      0.00      0.00
平均时间:         1      1077      0.00      0.00      0.00
```
runq-sz：运行队列的长度（等待运行的进程数）
plist-sz：进程列表中tasks的数量
ldavg-1：最后1分钟的系统平均负载（System load average）
ldavg-5：过去5分钟的系统平均负载
ldavg-15：过去15分钟的系统平均负载

**inode、文件和其他内核表信息监控**
```
[bhuser@bhqa ~]$ sar -v 2 2
Linux 2.6.32-504.8.1.el6.x86_64 (bhqa) 	2016年07月28日 	_x86_64_	(2 CPU)

01时01分44秒 dentunusd   file-nr  inode-nr    pty-nr
01时01分46秒    630020      6112    373635         1
01时01分48秒    630021      6112    373634         1
平均时间:    630020      6112    373634         1
```
dentunusd：目录高速缓存中未被使用的条目数量
file-nr：文件句柄（file handle）的使用数量
inode-nr：索引节点句柄（inode handle）的使用数量
pty-nr：系统pty（伪终端）使用数量

**网络设备信息监控**
```
[bhuser@bhqa ~]$ sar -n DEV 2 3
Linux 2.6.32-504.8.1.el6.x86_64 (bhqa) 	2016年07月28日 	_x86_64_	(2 CPU)

01时20分32秒     IFACE   rxpck/s   txpck/s    rxkB/s    txkB/s   rxcmp/s   txcmp/s  rxmcst/s
01时20分34秒        lo    380.83    380.83     22.92     22.92      0.00      0.00      0.00
01时20分34秒      eth0      7.25      9.33      0.79      0.92      0.00      0.00      0.00

01时20分34秒     IFACE   rxpck/s   txpck/s    rxkB/s    txkB/s   rxcmp/s   txcmp/s  rxmcst/s
01时20分36秒        lo    383.42    383.42     23.32     23.32      0.00      0.00      0.00
01时20分36秒      eth0     40.93     52.33     17.23      4.74      0.00      0.00      0.00

01时20分36秒     IFACE   rxpck/s   txpck/s    rxkB/s    txkB/s   rxcmp/s   txcmp/s  rxmcst/s
01时20分38秒        lo    365.98    365.98     21.63     21.63      0.00      0.00      0.00
01时20分38秒      eth0      7.22      8.25      1.29      1.06      0.00      0.00      0.00

平均时间:     IFACE   rxpck/s   txpck/s    rxkB/s    txkB/s   rxcmp/s   txcmp/s  rxmcst/s
平均时间:        lo    376.72    376.72     22.62     22.62      0.00      0.00      0.00
平均时间:      eth0     18.45     23.28      6.43      2.23      0.00      0.00      0.00
```
IFACE	网络设备名
rxpck/s	每秒接收的包总数
txpck/s	每秒传输的包总数
rxkB/s	每秒接收的字节（byte）总数
txkB/s  每秒传输的字节（byte）总数
rxcmp/s	每秒接收压缩包的总数
txcmp/s	每秒传输压缩包的总数
rxmcst/s	每秒接收的多播（multicast）包的总数

更详细的请查看[官方文档](http://sebastien.godard.pagesperso-orange.fr/man_sar.html)
