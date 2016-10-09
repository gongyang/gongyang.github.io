---
title : dstat--多功能系统资源统计生成工具
date: 2016-8-7
---
## dstat--多功能系统资源统计生成工具
dstat是一个用来替换vmstat、iostat、netstat、nfsstat和ifstat这些命令的工具，是一个全能系统信息统计工具，而且还可以通过使用插件扩展其功能。

### 安装
centos系统上直接使用yum安装
```
yum install dstat
```

### 使用
使用语法
```
dstat [-afv] [options..] [delay [count]]
```
安装完成之后，我们就可以直接使用dstat了，默认参数是-cdngy，分别代表显示cpu、disk、net、page、system信息，还可以在命令后面加上输出间隔时间(delay)以及输出次数(count)。
```
[root@bhqa ~]# dstat 3 5
----total-cpu-usage---- -dsk/total- -net/total- ---paging-- ---system--
usr sys idl wai hiq siq| read  writ| recv  send|  in   out | int   csw 
 10   4  85   1   0   0|  46k   40k|   0     0 |5085B 3034B| 599  2316 
  8   4  85   3   0   1|  24k  105k| 493B  965B|   0     0 |1744  2630 
 11   4  86   0   0   0|   0    23k|4642B 5476B|   0     0 |1653  2086 
  7   3  90   0   0   0|   0     0 | 551B  738B|   0     0 |1523  1969 
  7   3  90   0   0   0|   0    12k|1421B 1211B|   0     0 |1590  2010 
  8   3  88   0   0   1|  67k 1365B|1607B 1008B|   0     0 |1592  1975 
```
其中cpu一栏中的hiq、siq分别为硬中断和软中断次数。
system一栏的int、csw分别为系统的中断次数（interrupt）和上下文切换（context switch）。
其他常用选项
```
-c：显示CPU系统占用，用户占用，空闲，等待，中断，软件中断等信息。 
-C：当有多个CPU时候，此参数可按需分别显示cpu状态，例：-C 0,1 是显示cpu0和cpu1的信息。 
-d：显示磁盘读写数据大小。 
-D  和-C类似，默认是显示所有磁盘信息，使用-D可以指定查看单个磁盘的信息
-n：显示网络状态。 
-N eth1,total：有多块网卡时，指定要显示的网卡。 
-l：显示系统负载情况。 
-m：显示内存使用情况。 
-g：显示页面使用情况。 
-p：显示进程状态。 
-s：显示交换分区使用情况。 
-S：类似D/N。 
-r：I/O请求情况。 
-y：系统状态。 
--ipc：显示ipc消息队列，信号等信息。 
--socket：用来显示tcp udp端口状态。 
-a：此为默认选项，等同于-cdngy。 
-v：等同于 -pmgdsc -D total。 
--output 文件：此选项也比较有用，可以把状态信息以csv的格式重定向到指定的文件中，以便日后查看。例：dstat --output /tmp/dstat.csv
```
更多的参数请查看[man dstat](http://linux.die.net/man/1/dstat) 或者使用dstat -h来查看

除了常用的功能，dstat还支持插件提供而外功能。
通过输入命令dstat --list查看所有参数
上面internal是dstat本身自带的一些监控参数；
下面/usr/share/dstat中是dstat的插件，这些插件可以扩展dstat的功能，如可以监控电源（battery）、mysql等。
```
[root@bhqa ~]# dstat --list
internal:
	aio, cpu, cpu24, disk, disk24, disk24old, epoch, fs, int, int24, io, ipc, load, lock, mem, net, page, page24, 
	proc, raw, socket, swap, swapold, sys, tcp, time, udp, unix, vm
/usr/share/dstat:
	battery, battery-remain, cpufreq, dbus, disk-util, fan, freespace, gpfs, gpfs-ops, helloworld, innodb-buffer, 
	innodb-io, innodb-ops, lustre, memcache-hits, mysql-io, mysql-keys, mysql5-cmds, mysql5-conn, mysql5-io, 
	mysql5-keys, net-packets, nfs3, nfs3-ops, nfsd3, nfsd3-ops, ntp, postfix, power, proc-count, rpc, rpcd, 
	sendmail, snooze, thermal, top-bio, top-cpu, top-cputime, top-cputime-avg, top-io, top-latency, 
	top-latency-avg, top-mem, top-oom, utmp, vm-memctl, vmk-hba, vmk-int, vmk-nic, vz-cpu, vz-io, vz-ubc, wifi
```
例如通过--top-(io|bio|cpu|cputime|cputime-avg|mem) 这几个选项，可以看到具体是那个用户那个进程占用了相关系统资源，对系统调优非常有效。如查看当前占用I/O、cpu、内存等最高的进程信息可以使用dstat --top-mem --top-io --top-cpu。

