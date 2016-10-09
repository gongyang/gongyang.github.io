---
title : Monit
date: 2016-7-8
---
## Monit
[下载地址](https://mmonit.com/monit/dist/monit-5.18.tar.gz)

#### 什么是monit
monit是用来管理unix系统上面的进程、程序、文件、目录以及文件系统的实用工具。
monit能进行自动修复功能，比如对于没有运行的进程进行start操作，对于没有响应的进程进行restart操作，对于暂用过多资源的进程进行stop操作等。
monit还能监控文件、文件夹以及文件系统的一些变化。比如时间戳的变化、checksun的变化、以及文件大小的变化。
monit可以执行各种TCP/IP的网络检查。

#### 配置文件
monit的监控主要通过配置文件monitrc（5.1.4以前叫monit.conf），默认会读取~/.monitrc，如果没有，则会去读取/etc/monitrc，如果都没找到，就去读取~/monitrc。
当然，我们也可以在运行的时候指定配置文件地址
 例如：
```
 monit -c /var/monit/monitrc
```
配置文件分为Global section和Services

### Global section
先来看一下Global section需要设置的选项
```
set daemon  30  # 设置monit守护进程多长时间运行一次

set logfile /var/monit/monit.log #设置日志输出地址，默认输出到系统日志里面

set mailserver  localhost #设置邮件服务器地址

set eventqueue
    basedir /var/monit
    slots 100    #当邮件服务器出问题了，可以通过设置eventqueue，让事件存放在basdir里面，slots表示队列最多限制
    
set mail-format { from: monit@foo.bar } #设置邮件格式 from地址

set alert 442103349@qq.com   #报警接收地址，可设置多个；monit默认只发送一次失败邮件，如果想要发送多个可以加上参数“set alert 442103349@qq.com with reminder on 5 cycles”这样会5个失败周期发送一次报警；如果对于某些监控不需要报警可以这样
check process p1 with pidfile /var/run/p1.pid 
noalert 442103349@qq.com

set httpd port 2812 #开始http监控服务，可以通过web页面进行监控
    use address localhost  #如果不注释这一条，只能本地访问http服务，所以一般注释掉
    allow 127.0.0.1
    allow 192.168.1.0/24 
    allow admin:monit  #web页面登录帐号
    allow ggyy:gongyang read-only #设置帐号只有查看权限（很重要）
```

### Services
Services 主要设置具体监控项目
每个服务可以有相关联的启动、停止和重启方法，Monit可以使用它来执行服务行为。
```
check process sendmail with pidfile /var/run/sendmail.pid
   start program = "/etc/init.d/sendmail start" with timeout 60 seconds
   stop program = "/etc/init.d/sendmail stop"
```

monit是通过set daemon设定的固定周期检查，我们也可以设定几个周期检查一次
例：
```
check process nginx with pidfile /var/run/nginx.pid
       every 2 cycles
设定两个周期检查一次
```
```
check program checkOracleDatabase
        with path /var/monit/programs/checkoracle.pl
       every "* 8-19 * * 1-5"
每周1-5天的8-19点做检查，设置时间方法和crontab一样
```
```
check process mysqld with pidfile /var/run/mysqld.pid
       not every "* 0-3 * * 0"
星期天的凌晨0-3点不执行检查
```

#### service restart limit
monit对于restart一些限制机制
例如：
```
if 2 restarts within 3 cycles then unmonitor
如果三个周期内重启了两次，就不监控了
```
```
if 5 restarts within 5 cycles then exec "/tmp/1.sh"
 5个周期重启了5次，就执行某个脚本
```
```
if 7 restarts within 10 cycles then stop
10个周期重启了7次，就直接stop
```

#### monit tests
语法
```
 IF <test> THEN <action> [ELSE IF SUCCEEDED THEN <action>]
```
action有以下几种

- ALERT
- start
- stop
- restart
- exec
- UNMONITOR

例如：
```
if failed
    port 80
    for 3 cycles
 then alert
```
```
check filesystem rootfs with path /dev/hda1
  if space usage > 80% for 5 times within 15 cycles then alert
  if space usage > 90% for 5 cycles then exec '/try/to/free/the/space'
```

**是否存在监控**
```
IF [DOES] NOT EXIST THEN <action>
```

```
check file with path /cifs/mydata
       if does not exist then exec "/usr/bin/mount_cifs.sh"
```

**使用资源监控**
```
if cpu is greater than 50% for 5 cycles then restart
```

**checksum 监控**
```
check file passwd with path /etc/password
if failed
    checksum expect 8f7f419955cefa0b33a2ba316cba3659
 then alert
```
```
check file apache_conf with path /etc/apache/httpd.conf
     if changed checksum then alert
```

**时间戳监控**
```
check file passwd with path /etc/password
   if changed timestamp then alert
```
```
check file passwd with path /etc/password
   if changed timestamp > 2 MINUTE then alert
```

**文件大小监控**
```
 check file mydb with path /data/mydatabase.db
       if size > 1 GB then alert
```

**文件内容监控**
```
check file syslog with path /var/log/syslog
        ignore content = "^monit"
        if content = "^mrcoffee" then alert
```

**磁盘空间监控**
```
check filesystem rootfs with path /
       if space usage > 90% then alert
```

**inode 监控**
```
check filesystem rootfs with path /
       if inode usage > 90% then alert
```

**权限监控**
```
 check file shadow with path /etc/shadow
       if failed permission 0640 then alert
```

**uid gid  pid ppid监控**
```
 check file shadow with path /etc/shadow
       if failed uid root then alert
```

```
check file shadow with path /etc/shadow
       if failed gid shadow then alert
```

```
check process sshd with pidfile /var/run/sshd.pid
       if changed pid then alert
```

```
check process myproc with pidfile /var/run/myproc.pid
       if changed ppid then exec "/my/script"
```

**运行时间监控**
```
check process myapp with pidfile /var/run/myapp.pid
    start program = "/etc/init.d/myapp start"
    stop program = "/etc/init.d/myapp stop"
    if uptime > 3 days then restart
```

**网络链接状态监控**
```
 check network eth0 with interface eth0
       if failed link then alert
```
```
check network eth0 with interface eth0
       start program = '/sbin/ipup eth0'
       stop program = '/sbin/ipdown eth0'
       if failed link then restart
```

**网络带宽监控**
```
check network eth0 with interface eth0
       if upload > 500 kB/s then alert
       if total download > 1 GB in last 2 hours then alert
       if total download > 10 GB in last day then alert
```

**网络数据包监控**
```
 check network eth0 with interface eth0
       if upload > 1000 packets/s then alert
       if total upload > 900000 packets in last hour then alert
```

配置完成之后使用-t参数检查一遍语法
```
monit -t
```
启动
直接运行monit即可
```
monit
```
可以通过status参数查看监控项目
```
monit status
```

