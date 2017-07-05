---
title : linux内建命令之--ulimit
date: 2016-10-27
---
## linux内建命令之--ulimit
ulimit是linux内建指令之一，主要用于控制shell程序的资源，通过限制资源使用来改善系统性能。

### 内部命令  外部命令
可能有的同学对于内建命令这个概念有些陌生，我在这里简单的讲解下：
linux命令分为内部命令（内建指令）和外部命令，内部命令是在系统启动的时候就加载到内存中，并常驻内存，执行效率高；外部命令是系统程序部分，用户使用的时候从硬盘中读入内存。
如何区分？
我们在命令前面加上type来查看这个命令是内部命令，还是外部命令。
```
[ggyy@gy-vm02 ~]$ type ulimit
ulimit is a shell builtin
[ggyy@gy-vm02 ~]$ type mysql
mysql is /usr/bin/mysql
[ggyy@gy-vm02 ~]$ type type
type is a shell builtin
```
内部命令用户输入时系统调用的速率快，不是内置命令，系统将会读取环境变量文件.bash_profile、/etc/profile去找PATH路径。

**tip**
使用help命令方式
内部命令
```
help command
```
外部命令
```
command help
```

### ulimit的功能
ulimit 用于限制 shell 启动进程所占用的资源，具体有哪些我们可以可以使用help ulimit详细的查看。查看当前shell的ulimit设置，使用ulimit -a进行查看
```
[ggyy@gy-vm02 ~]$ ulimit -a
core file size          (blocks, -c) 0
data seg size           (kbytes, -d) unlimited
scheduling priority             (-e) 0
file size               (blocks, -f) unlimited
pending signals                 (-i) 30493
max locked memory       (kbytes, -l) 64
max memory size         (kbytes, -m) unlimited
open files                      (-n) 1024
pipe size            (512 bytes, -p) 8
POSIX message queues     (bytes, -q) 819200
real-time priority              (-r) 0
stack size              (kbytes, -s) 10240
cpu time               (seconds, -t) unlimited
max user processes              (-u) 1024
virtual memory          (kbytes, -v) unlimited
file locks                      (-x) unlimited
```
如果你没有调整过，那么看到的数据应该和我的是一样的（centos 6），这也是系统默认的设置。
一般的服务器使用默认的配置就足够了，如果你的服务器是对外的web server，或者是db server，比如说nginx服务器、mongodb服务器、mysql服务器等，对于这类服务器，你可能需要对其中某几项参数做下调整。其中主要是open files和max user processes这两项参数。
```
open files ：可以打开最大文件描述符的数量
max user processes：最大生成的线程数
```

### ulimit 使用
ulimit 命令的格式为：
```
ulimit: ulimit [-SHacdefilmnpqrstuvx] [limit]
```

比如说，我要设置当前shell最大打开文件数为2000：

```
[ggyy@gy-vm02 ~]$ ulimit -n 2000
[ggyy@gy-vm02 ~]$ ulimit -n
2000
```

设置最大生成线程数为10000

```
[ggyy@gy-vm02 ~]$ ulimit -u 10000
[ggyy@gy-vm02 ~]$ ulimit -u 
10000
```

**在shell里面通过command配置的参数，只对当前的shell起作用，如果你这个时候退出，然后重新打开shell，你的这个设置就失效了。**

这个时候我们打开另外一个shell测试一下；
```
[ggyy@gy-vm02 ~]$ ulimit -n
1024
[ggyy@gy-vm02 ~]$ ulimit -u
1024
```
发现还是系统默认值。

如果你想永久保存设置，对所有的 terminal都生效，你可以把配置写在 /etc/security/limits.conf 或者是 /etc/security/limits.d/*.conf。

### ulimit配置文件

limits.conf的格式如下： 
```
<domain>  <type>  <item>  <value>
```
- domain ：设置需要被限制的用户名，组名前面加@和用户名区别。也可以用通配符*来做所有用户的限制。
- type ： 有 soft，hard 和 -
soft 指的是当前系统生效的设置值
hard 表明系统中所能设定的最大值
soft 的限制不能比har 限制高
用 - 就表明同时设置了 soft 和 hard 的值。
- resource ： 
core - 限制内核文件的大小
date - 最大数据大小
fsize - 最大文件大小
memlock - 最大锁定内存地址空间
nofile - 打开文件的最大数目
rss - 最大持久设置大小
stack - 最大栈大小
cpu - 以分钟为单位的最多 CPU 时间
noproc - 进程的最大数目
as - 地址空间限制
maxlogins - 此用户允许登录的最大数目
- value ： 需要设置的值

其中关于type我详细解释一下：
type分为soft和hard，也就是软限制和硬限制。怎么理解：
软限制：我们在shell中使用命令进行设置的就是软限制，它是可以通过命令设置动态调整的，但是不能超过硬限制设置的值。
硬限制：就是该参数可设置的最大值

我们举个例子，我们刚刚使用ulimit -n设置当前shell最大打开文件为2000，那我们那这个值改成10000试试：
```
[ggyy@gy-vm02 ~]$ ulimit -n 10000
-bash: ulimit: open files: cannot modify limit: Operation not permitted
[ggyy@gy-vm02 ~]$ ulimit -n
2000
```
设置不成功，当前的值还是2000，之所以不能设置，是因为我们还有一个硬限制，使用 -H 参数查看硬限制的值
```
[ggyy@gy-vm02 ~]$ ulimit -H -n
4096
```
也就是说，软限制最大值不能超过4096。

### 总结
ulimit 作为对资源使用限制的一种工作，是有其作用范围的。ulimit 限制的是当前 shell 进程以及其派生的子进程。举例来说，如果用户同时运行了两个 shell 终端进程，只在其中一个环境中执行了 ulimit – s 100，则该 shell 进程里创建文件的大小收到相应的限制，而同时另一个 shell 终端包括其上运行的子程序都不会受其影响。
我们可以查看**cat /proc/pid/limits**该进程的资源限制情况。
如：
```
[root@gy-vm02 ~]# cat /proc/2841/limits 
Limit                     Soft Limit           Hard Limit           Units     
Max cpu time              unlimited            unlimited            seconds   
Max file size             unlimited            unlimited            bytes     
Max data size             unlimited            unlimited            bytes     
Max stack size            10485760             unlimited            bytes     
Max core file size        0                    unlimited            bytes     
Max resident set          unlimited            unlimited            bytes     
Max processes             14779                14779                processes 
Max open files            1024                 4096                 files     
Max locked memory         65536                65536                bytes     
Max address space         unlimited            unlimited            bytes     
Max file locks            unlimited            unlimited            locks     
Max pending signals       14779                14779                signals   
Max msgqueue size         819200               819200               bytes     
Max nice priority         0                    0                    
Max realtime priority     0                    0                    
Max realtime timeout      unlimited            unlimited            us 
```

