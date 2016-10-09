---
title: Docker 入门系列之--容器
date: 2016-1-31
---
对于Docker的安装，我这里就不专门细说了，官方有详细的安装说明，各个系统都有示例，[Docker安装][install]

上一遍文章我有讲到，容器主要是用来运行进程实例的，那么我们如何来运行容器，管理容器了，请您接着往下读。

首先我们查看Docker是否成功安装，我们在命令行界面输入
docker info
```
[root@gy-vm06 ~]# docker info
Containers: 8
Images: 58
Storage Driver: devicemapper
 Pool Name: docker-253:0-67837516-pool
 Pool Blocksize: 65.54 kB
 Backing Filesystem: xfs
 Data file: /dev/loop0
 Metadata file: /dev/loop1
 Data Space Used: 3.963 GB
 Data Space Total: 107.4 GB
 Data Space Available: 35.1 GB
 Metadata Space Used: 4.977 MB
 Metadata Space Total: 2.147 GB
 Metadata Space Available: 2.143 GB
 Udev Sync Supported: true
 Deferred Removal Enabled: false
 Data loop file: /var/lib/docker/devicemapper/devicemapper/data
 Metadata loop file: /var/lib/docker/devicemapper/devicemapper/metadata
 Library Version: 1.02.107-RHEL7 (2015-10-14)
Execution Driver: native-0.2
Logging Driver: json-file
Kernel Version: 3.10.0-327.el7.x86_64
...
```
从信息中我们可以看到容器和镜像的数量，以及docker的基本配置等。如果你在输入docker info的时候出现下面的情况
```
[ggyy@gy-vm06 ~]$ docker info
Get http:///var/run/docker.sock/v1.19/info: dial unix /var/run/docker.sock: permission denied. Are you trying to connect to a TLS-enabled daemon without TLS?
```
说明你不是root用户；docker的守护进程是用root身份运行的，因为他要来处理普通用户无法完成的操作（如文件系统的挂载）。docker命令是Docker守护进程的客户端程序，同样也需要root身份运行。

###接下来我们来运行第一个容器

首先来看一下我的宿主机操作系统以及内核版本
```
[root@gy-vm06 ~]# uname -r
3.10.0-327.el7.x86_64
[root@gy-vm06 ~]# cat /etc/redhat-release 
CentOS Linux release 7.2.1511 (Core) 
[root@gy-vm06 ~]# 
```
我的操作系统是centos 7.2 内核版本是3.10，均达到了Docker官方的推荐要求

然后，运行我们的第一个容器
```
[root@gy-vm06 ~]# docker run -i -t centos /bin/bash
Unable to find image 'centos:latest' locally
Trying to pull repository docker.io/library/centos ... latest: Pulling from library/centos
838c1c5c4f83: Pull complete 
5764f0a31317: Pull complete 
60e65a8e4030: Pull complete 
60e65a8e4030: Pulling fs layer 
library/centos:latest: The image you are pulling has been verified. Important: image verification is a tech preview feature and should not be relied on to provide security.


Digest: sha256:8072bc7c66c3d5b633c3fddfc2bf12d5b4c2623f7004d9eed6aae70e0e99fbd7
Status: Downloaded newer image for docker.io/centos:latest

[root@1ab6fed17c56 /]#
```
关于docker的参数说明，你可以查看[官方文档][help]，你也可以使用docker help获取命令帮助，你还可以执行man docker-run。

刚刚的命令行的内容以及输出内容，我们来一一分析下。
docker run命令是用来创建容器的，-i 和-t是两个命令行参数。-i是表示容器中的标准输入STDIN是开启的，-t则是告诉Docker为要创建的容器分配一个伪tty终端。这样，新创建的容器才能提供一个交互式的shell。紧接着后面的参数是告诉Docker基于什么镜像来创建容器，例子中使用的centos镜像。像 ubuntu、centos这类镜像叫做基础（base）镜像，由Docker公司提供，保存在[Docker Hub][hub] Registry上。

当我们执行了这条命令，Docker首先会检查本地是否有centos镜像存在，如果没有，就去直接去Docker Hub Registry上面去找，一旦找到，就会下载该镜像，并保存到本地。
随后，Docker在文件系统内部用这个镜像创建了一个容器，还记得上一篇文章里面有说过，创建容器，实际上是在该镜像的最上层加了一层可读写层，更具体的细节，会在后面讲镜像的时候来细致分析。该容器拥有自己的网络、IP地址，以及一个用来和宿主机进行通信的桥接网络接口。最后，我们告诉Docker要在容器中运行/bin/bash命令。

###使用第一个容器

创建容器之后我们就已经进入到了容器里面，我们运行hostname看看
```
[root@1ab6fed17c56 /]# hostname
1ab6fed17c56
```
1ab6fed17c56就是我们的主机名。
再查看一下ip
```
[root@1ab6fed17c56 /]# ip a
bash: ip: command not found
```
发现没有这条命令，看来需要自己安装了
```
[root@1ab6fed17c56 /]# yum install iproute
```
安装完成之后再次查看
```
[root@1ab6fed17c56 /]# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN 
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
18: eth0@if19: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP 
    link/ether 02:42:ac:11:00:03 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 172.17.0.3/16 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::42:acff:fe11:3/64 scope link 
       valid_lft forever preferred_lft forever
```
我们看到容器的内网ip是172.17.0.3，关于容器之间的ip以及通讯，我在后面会详细讲解。
完成操作之后，输入exit，就可以退出容器，回到我们的宿主机上面来。当我们退出容器之后，容器也就停止了运行，只有在指定的/bin/bash命令处于运行状态的时候，我们容器才会相应的处于运行状态。一旦退出容器，bash命令也就结束了，这时容器也随之停止了。
我们可以通过docker ps -a 查看当前系统中所有创建了的容器。
```
[root@gy-vm06 ~]# docker ps -a
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS                        PORTS               NAMES
d8de2a7f00c9        centos              "/bin/bash"         41 minutes ago      Exited (127) 41 minutes ago                       sleepy_yalow
3b99bef09208        ubuntu              "/bin/bash"         48 minutes ago      Exited (127) 46 minutes ago                       happy_lalande
1ab6fed17c56        centos              "/bin/bash"         2 hours ago         Up 38 minutes                                     berserk_fermi
0da6c3122510        ubuntu              "/bin/bash"         2 hours ago         Exited (0) 2 hours ago                            ecstatic_aryabhata
97c208baae8f        715dfae8b624        "/bin/bash"         2 days ago          Exited (137) 35 hours ago     8080/tcp            adoring_swartz
```
可以看到容器ID 1ab6fed17c56，其实hostname也就是容器ID，由于我没有退出容器，我是在另外一个terminal上面运行的docker ps -a，所以可以看到当前容器是UP状态。其他的都是Exited的状态。默认docker ps是显示正在运行的容器列表：
```
[root@gy-vm06 ~]# docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
1ab6fed17c56        centos              "/bin/bash"         2 hours ago         Up 43 minutes                           berserk_fermi
```
通过docker ps我们可以看到容器的id，来至哪个镜像，运行什么命令，创建时间，状态，打开了哪些端口，以及容器的名称。
有两种方式可以指定唯一容器，CONTAINER ID和NAMES。在这里，容器ID就是1ab6fed17c56，容器的名称就是berserk_fermi。这里由于我们在创建容器的时候没有指定容器的名字，Docker随机分配了一个名字，我们也可以在创建容器的时候加上 --name test_name 这样来给容器命名。容器的命名规则只能是一下字符：小写字母a-z、大写A-Z、数字0-9、下划线、点、横线，也就是[a-zA-Z0-9_.-]。

我们可以使用docker start 和docker stop来启动，停止容器。还可以用docker kill快速停止容器。
```
[root@gy-vm06 ~]# docker stop 1ab6fed17c56
1ab6fed17c56
[root@gy-vm06 ~]# docker start 1ab6fed17c56
1ab6fed17c56
[root@gy-vm06 ~]# docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
1ab6fed17c56        centos              "/bin/bash"         3 hours ago         Up 4 seconds                            berserk_fermi
```
启动容器之后，我们可以使用docker attach附着到容器上去
```
[root@gy-vm06 ~]# docker attach 1ab6fed17c56

[root@1ab6fed17c56 /]# 
```
这里有个细节需要注意一下，需要敲一下回车，才可以进入该会话。

如果想要创建守护式进程，则是运行docker run -d来创建
```
[root@gy-vm06 ~]# docker run -di --name ggyy centos /bin/bash
a6171f4f68a5afc7dba12f4b88c04d78b816f12c3f5ccd005989e5531e309777
[root@gy-vm06 ~]# 
[root@gy-vm06 ~]# docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED              STATUS              PORTS               NAMES
a6171f4f68a5        centos              "/bin/bash"         6 seconds ago        Up 5 seconds                            ggyy
```
这样就创建了一个后台运行的容器ggyy
我们还可以通过docker top ggyy来查看容器内部运行的进程
```
[root@gy-vm06 ~]# docker top ggyy
UID                 PID                 PPID                C                   STIME               TTY                 TIME                CMD
root                4400                3819                0                   21:25               ?                   00:00:00            /bin/bash
```
如果我们想要在后台运行的容器里面再启动其他进程，或者执行一些操作命令，该怎么办了，在Docker1.3之后，我们可以通过docker exec命令来操作。
比如说我需要进行一个交互式的shell，那就这样
```
[root@gy-vm06 ~]# docker exec -it ggyy /bin/bash
[root@a6171f4f68a5 /]# ps
  PID TTY          TIME CMD
    6 ?        00:00:00 bash
   17 ?        00:00:00 ps
```
这样当你退出这个容器的时候，容器还是运行的
```
[root@a6171f4f68a5 /]# exit
exit
[root@gy-vm06 ~]# 
[root@gy-vm06 ~]# docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
a6171f4f68a5        centos              "/bin/bash"         9 minutes ago       Up 9 minutes                            ggyy
52ab3bc1a050        centos              "/bin/bash"         11 minutes ago      Up 11 minutes                           furious_mahavira
```
你也可以执行一些后台任务
```
[root@gy-vm06 ~]# docker exec -d ggyy touch /etc/ggyy.conf
[root@gy-vm06 ~]# docker exec -it ggyy /bin/bash
[root@a6171f4f68a5 /]# ls /etc/ggyy.conf 
/etc/ggyy.conf
```
有了docker exec，我们可以在正在运行的容器中进行维护、监控以及任务管理。

在有些情况下面，由于某种错误导致容器的停止运行，我么还可以添加参数--restart让容器自动重启，--restart标志会检查容器的退出代码，并据此来决定是否要重启容器。默认情况下Docker是不会重启容器的。
```
 docker run  --restart=always --name=gy-ban -di centos /bin/bash
```
在上面的代码中，--restart的标志设置为always。无论容器的退出代码是什么（容器的退出代码可以通过docker ps -a 中STATUS一栏查看），Docker都会自动重启该容器。除了always，我们还可以使用on-failure，这样，只有当容器的退出代码为非0值的时候，才会自动重启（0是正常退出）。而且on-failure还可以加参数，表示重启的次数，如下面的代码
```
 --restart=on-failure:5
```
这样，当容器的退出代码为非0时，Docker会尝试自动重启最多5次。

更多的容器信息可以使用docker inspect来查看，加上参数--format或者-f可以选定查看结果
```
[root@gy-vm06 ~]# docker inspect -f '{{ .NetworkSettings.IPAddress }}' ggyy
172.17.0.7
```
-f 后面的参数是Go语言模版。

最后，删除某个容器使用docker rm 就可以了，但是需要注意一下的就是，运行中的容器是不能删除的，你可以stop或者kill来停止之后，才能删除。
如果想要一次删除多个容器可以使用如下代码
```
docker rm `docker ps -a -q`
```
好了关于docker容器的使用，就先讲到这里。

由于博客系统刚刚搭建，评论功能还没有时间去弄，所以大家对于我的博客中有什么错误的地方，或者写得不好的地方，可以往我的邮箱里面提，我的邮箱是
gongyang314@163.com
海纳百川，有容乃大！希望有什么不足的，大伙帮我来改进。






[hub]:https://hub.docker.com/
[help]:https://docs.docker.com/engine/reference/commandline/cli/
[install]:https://docs.docker.com/

---

