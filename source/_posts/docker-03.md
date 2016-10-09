---
title: Docker 入门系列之--镜像
date: 2016-2-21
---

上一篇我们已经讲过了，docker容器的创建和管理；今天，我们来讲一讲docker镜像的创建，以及管理。
我们知道，容器是基于镜像生成的。那么，什么是docker镜像了？
根据docker官方文档的描叙，Image（镜像）是docker术语的一种，代表一个只读的layer。而layer则具体代表docker container文件系统中可叠加的部分。
被我这么一介绍，相信很多小伙伴们更加迷惑了，怎么又出来一个layer，什么叫做可叠加的文件系统。在解答这些疑惑之前，让我们来了解几个和docker镜像相关的几个概念：rootfs、union mount。

### rootfs
rootfs：代表一个docker container在启动时其内部进程可见的文件系统视角，或者说是container的根目录。当然，该目录下面还有container需要的系统文件，容器文件等。
说到rootfs，相信大家不会陌生，因为在linux操作系统内核启动的时候，内核会先挂载一个只读的rootfs，然后在系统自检完成之后，将其改为read-write，这样我们就可以在rootfs上面进行读写操作了。但是docker镜像却不这样，它在bootfs自检完成之后，并不会把rootfs的read-only改成read-write，而是利用union mount（Union FS的一种挂在机制）将一个或多个read-only的rootfs加载到之前的read-only的rootfs上面。虽然加载多层的rootfs，但是在外面看来，仍像是一个文件系统。`在Docker体系里面，把union mount的这些read-only的rootfs叫做镜像` 细心的读者可能会问，此时的每一层rootfs都是read-only状态，如何进行写操作了。当我们创建容器的时候，也就是将docker镜像实例化的时候，系统再次使用union mount在一层或多层read-only的rootfs之上挂载一个read-write的文件系统，但是此时的read-write文件系统是一个空的文件系统。

### union mount
 union mount 是一种文件系统的挂载方式，允许同一时刻多个不同的文件系统挂在在一起，并且以一种文件系统的形式，将多个文件系统的内容合并成一个目录。
 一般情况下面，通过一种文件系统挂在内容至挂载点，挂载点原先的内容会被覆盖掉。但是如果是union mount则不会将挂载点目录中的内容隐藏，反而是将挂载点目录中的内容和被挂载的内容合并，并为合并后的内容提供一个统一独立的文件系统视角。通常来讲，被合并的文件系统中只有一个会以读写（read-write）模式挂载，而其他的文件系统的挂载模式均为只读（read-only）。实现这种Union mount技术的文件系统一般被称为Union Filesystem，较为常见的有UnionFS、AUFS、OverlayFS等。
 Aufs是Docker最初采用的文件系统，由于Aufs未能加入到Linux内核，考虑到兼容性问题，加入了Devicemapper的支持。目前，除少数版本如Ubuntu，Docker基本运行在Devicemapper基础上。

好了，关于概念性的就先介绍到这里。接下来，我们进入实战环节。

### 常用命令
我们在运行容器的时候，选择的镜像都是已经下载到本地了，如果你在启动容器的时候，选择的镜像本地没有，那么docker会先自动进行下载。本地存放的目录一般是在/var/lib/docker下面。

**列出本地镜像**
docker images	
```
[root@gy-vm06 ~]# docker images
REPOSITORY               TAG                 IMAGE ID            CREATED             VIRTUAL SIZE
ggyy/one                 latest              c4ff03bdb0d8        4 days ago          1.187 GB
ggyy/ruby                2.1.5               f68ff1ea510c        5 days ago          946.4 MB
daocloud.io/gyban/ruby   2.1.5               f68ff1ea510c        5 days ago          946.4 MB
daocloud.io/mysql        5.7                 596847483ae2        3 weeks ago         360.2 MB
daocloud.io/rails        onbuild             26e7aa58d0c5        9 weeks ago         772.1 MB
daocloud.io/centos       6                   3bbbf0aca359        4 months ago        190.6 MB
```
通过docker images可以看到镜像来至于哪个仓库，它的tag，镜像的UUID，创建时间和大小。
镜像是从仓库下载下来的，镜像是保存在仓库中，而仓库存在于Registy中。默认的Registry是由docker公司自己运营的[Docker Hub](https://hub.docker.com/)。一个仓库里面可以存放多个镜像，例如centos6.5和centos6.6都是存放在centos这个仓库里面，所以需要使用tag以示区分。我们可以在仓库名后面加上一个冒号和标签名来指定该仓库中的某一镜像，如果不指定，默认使用latest。
Docker Hub中有两种类型的仓库：用户仓库和顶层仓库。用户仓库的镜像都是由Docker用户创建的，而顶层仓库则是由Docker内部的人来管理的。用户仓库的命令由用户名和仓库名两部分组成。

下面两个例子，第一个没有指定tag，则使用默认的latest镜像启动容器，第二个指定了tag，则会使用centos6.5版本启动容器。
```
[root@gy-vm06 ~]# docker run -it centos /bin/bash
```
```
[root@gy-vm06 ~]# docker run -it centos:6.5 /bin/bash
```

**拉取镜像**
虽然在运行容器的时候，docker会自动下载镜像，但我们也可以提前将镜像拉取下来。
```
docker pull centos
```
上面这条命令会将整个centos仓库里面的镜像都拉取下来，如果只需要拉取某个镜像使用
```
docker pull centos:6.5
```

**查找镜像**
如果你想知道某个应用在docker hub上面有没有标准镜像，你可以使用
```
docker search nginx
```

**构建镜像**
前面我们所使用的镜像，都是制作好了的，那么我们如何制作按照自己的需求来定制化一些镜像了。
构建Docker镜像有一下两种方法。

- 使用docker commit 命令

- 使用docker build命令和Dockerfile文件

我们先用这两种方式来构建一个镜像，然后我们再对比这两种方式的利和弊

***docker commit***
docker commit 有点类似于版本控制系统里面的提交变更。我们先创建一个容器，并在容器里面做出修改，你可以创建一个文件，或者安装一个软件，最后再将修改提交为一个新镜像。
首先我们先运行一个容器，并且touch一个文件abc.txt
```
[root@gy-vm06 ~]# docker run -it daocloud.io/centos:6 /bin/bash
[root@a5886fb30e16 /]# touch abc.txt
[root@a5886fb30e16 /]# ls
abc.txt  dev  home  lib64       media  opt   root  sbin     srv  tmp  var
bin      etc  lib   lost+found  mnt    proc  run   selinux  sys  usr
[root@a5886fb30e16 /]# 
```
如果我们想保存当前状态，不想每次都创建abc.txt这个文件，我们需要使用exit命令从容器里面退出，之后使用docker commit命令进行提交。
操作如下：
```
[root@a5886fb30e16 /]# exit
exit
[root@gy-vm06 ~]# docker ps -l
CONTAINER ID        IMAGE                  COMMAND             CREATED             STATUS                     PORTS               NAMES
a5886fb30e16        daocloud.io/centos:6   "/bin/bash"         7 minutes ago       Exited (0) 7 seconds ago                       clever_swartz
[root@gy-vm06 ~]# docker commit a5886fb30e16 gongyang/abc
c1e704fc7820ede7b6e0a008e57be948486144d0d5d07bb784425880d0a1bcfb
[root@gy-vm06 ~]#
```
使用docker commit之前，首先使用docker ps  -l 获取刚刚运行的容器的信息，-l参数表示显示最后创建的一个容器信息。由于这个镜像是我个人创建的，所以是由用户名和仓库名来命名（gongyang/abc）。值得注意的是，docker commit提交的只是创建容器的镜像与容器当前状态之间有差异的部分（有兴趣的同学，可以使用docker save将原始镜像和commit之后的镜像进行对比），这使得该更新非常轻量。
然后我们使用docker images 查看刚刚创建的镜像
```
[root@gy-vm06 ~]# docker images
REPOSITORY               TAG                 IMAGE ID            CREATED             VIRTUAL SIZE
gongyang/abc             latest              c1e704fc7820        14 minutes ago      190.6 MB
ggyy/one                 latest              c4ff03bdb0d8        4 days ago          1.187 GB
ggyy/ruby                2.1.5               f68ff1ea510c        6 days ago          946.4 MB
daocloud.io/gyban/ruby   2.1.5               f68ff1ea510c        6 days ago          946.4 MB
daocloud.io/mysql        5.7                 596847483ae2        3 weeks ago         360.2 MB
daocloud.io/rails        onbuild             26e7aa58d0c5        9 weeks ago         772.1 MB
daocloud.io/centos       6                   3bbbf0aca359        4 months ago        190.6 MB
```
可以看到原始镜像daocloud.io/centos:6和创建后的镜像gongyang/abc大小是一样的，因为我只是touch了一个空文件。
接下来我们使用新镜像来运行一个容器
```
[root@gy-vm06 ~]# docker run -it gongyang/abc /bin/bash
[root@bc4a8a055ea0 /]# ls
abc.txt  dev  home  lib64       media  opt   root  sbin     srv  tmp  var
bin      etc  lib   lost+found  mnt    proc  run   selinux  sys  usr
[root@bc4a8a055ea0 /]#
```
可以看到，新运行的容器里面就有了abc.txt这个文件了，也就是说最开始容器的状态保存了下来。

***Dockerfile***
Docker公司并不推荐使用docker commit的方法来构建镜像。相反，推荐使用Dockerfile定义文件和docker build命令来构建镜像。Dockerfile使用基本的基于DSL语法的指令来构建一个Docker镜像，之后使用docker build命令基于该Dockerfile中的指令构建一个新的镜像。
第一个Dockerfile
```
[root@gy-vm06 ~]# mkdir dockerfile
[root@gy-vm06 ~]# cd dockerfile/
[root@gy-vm06 dockerfile]# touch Dockerfile
```
我们创建了一个dockerfile的目录来保存Dockerfile，这个目录就是我们的构建环境，Docker则称此环境为上下文或者构建上下文。Docker会在构建镜像时将构建上下文和该上下文中的文件和目录上传到Docker守护进程。这样Docker守护进程就能直接访问你想在镜像中存储的任何代码、文件或者其他数据。
```
#version 0.0.1
FROM daocloud.io/centos:6
MAINTAINER gongyang@boohee.com
RUN yum install -y nginx
EXPOSE 80
```
上面的Dockerfile由一系列的指令和参数组成。每条指令如FROM，都必须是大写字母，后面要跟一个参数：daocloud.io/centos:6。Dockerfile中的指令会按顺序从上倒下执行。
每条指令都会创建一个新的镜像层并对镜像进行提交。大体流程如下：

- docker从基础镜像运行一个容器
- 执行一条指令，对容器做出修改。
- 执行类似docker commit的操作，提交一个新的镜像层。
- docker再基于刚提交的镜像运行一个新容器
- 执行Dockerfile中的下一条指令，直到所有指令都执行完毕。

***Dockerfile常用指令***
除了上文中提到的RUN和EXPOSE。Dockerfile还提供更多其他指令。我们可以
在http://docs.docker.com/reference/builder查看关于Dockerfile全部指令的清单。
1、CMD
CMD指令用于指定一个容器启动时要运行的命令。这有点儿类似于RUN指令，只是RUN指令是指定镜像被构建时要运行的命令，而CMD是指定容器被启动时要运行的命令。这个和使用docker run命令启动容器时指定运行的命令/bin/bash命令非常类似。但是这里需要大伙注意一点，使用docker run 命令可以覆盖CMD指令。如果我们在Dockerfile里指定了CMD指令，而同时在docker run 命令行中也指定了要运行的命令，命令行中指定的命令会覆盖Dockerfile中的CMD指令。
我们来看一个简单的例子：
我们使用docker history命令查看一个镜像的创建过程
```
[root@gy-vm06 ~]# docker history daocloud.io/centos:6 
IMAGE               CREATED             CREATED BY                                      SIZE                COMMENT
3bbbf0aca359        4 months ago        /bin/sh -c #(nop) CMD ["/bin/bash"]             0 B                 
fea77d2fd61e        4 months ago        /bin/sh -c #(nop) LABEL License=GPLv2           0 B                 
91e6f84b8fe8        4 months ago        /bin/sh -c #(nop) LABEL Vendor=CentOS           0 B                 
2c2557968d48        4 months ago        /bin/sh -c #(nop) ADD file:c8f5f9054c3914e848   190.6 MB            
47d44cb6f252        5 months ago        /bin/sh -c #(nop) MAINTAINER The CentOS Proje   0 B 
```
这是用Dockerfile build出来的一个镜像，其中有定义CMD命令为/bin/bash，也就是说，如果我们使用该镜像启动的容器，如果不指定运行的命令，就会默认使用CMD里面定义的命令。
下面的例子，我们虽然没有指定运行的命令，但我们还是进入到了改容器的bash交互界面。
```
[root@gy-vm06 ~]# docker run -it daocloud.io/centos:6
[root@3dc66dd7d1af /]#
```
如果我们指定其它命令，则会覆盖CMD里面指定的命令
```
[root@gy-vm06 ~]# docker run -it daocloud.io/centos:6 /bin/ps
  PID TTY          TIME CMD
    1 ?        00:00:00 ps
```

**tip：**
在Dockerfile中只能指定一条CMD指令。如果指定了多条CMD指令，也只有最后一条CMD指令会被使用。

2、ENTRYPOINT
ENTRYPOINT和CMD指令非常类似，主要区别在于，CMD指令可以被docker run命令行中命令所覆盖，而ENTRYPOINT指令提供的命令则不容易在启动容器时被覆盖。并且还可以接受docker run命令行中指定的参数。
其实，ENTRYPOINT指令指定的命令也是可以被覆盖的，在docker run命令行中加上--entrypoint标志覆盖ENTRYPOINT指令

3、WORKDIR
WORKDIR指令用来指定容器创建时的工作目录，CMD和ENTRYPOINT指定的程序会在这个目录下执行。
我们可以使用该指令为Dockerfile中后续的一系列指令设置工作目录，也可以为最终的容器设置工作目录。
如：
```
WORKDIR /var/apps/one
RUN gem install rails
WORKDIR /usr/local/nginx/sbin
RUN nginx
```
上面例子中，首先将工作目录切换到/var/apps/one，安装rails，然后又将工作目录切换到/usr/local/nginx/sbin 启动nginx
我们也可以在docker run使用-w参数指定工作目录
```
[root@gy-vm06 ~]# docker run -it -w /var/log daocloud.io/centos:6  pwd
/var/log
```
该参数会将容器内的工作目录设置为/var/log

4、ENV
ENV指令用来在镜像构建过程中设置环境变量。
```
ENV RVM_PATH /home/rvm/
RUN gem install unicorn
```
该指令会以下面的方式执行
```
RVM_PATH=/home/rvm gem install unicorn
```
我们也可以在其他指令中直接使用这些环境变量
```
ENV RVM_PATH /home/rvm/
WORKDIR $RVM_PATH
```
这些环境变量会被持久保存到从我们镜像创建的任何容器中。
我们也可以在docker run命令行中使用-e来指定环境变量，不过这些变量将只会在运行时有效。
```
[root@gy-vm06 ~]# docker run -it -e "WEB_PORT=9999" daocloud.io/centos:6  env
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
HOSTNAME=b9d880f3e7e5
TERM=xterm
WEB_PORT=9999
HOME=/root
```
我么可以看到WEB_PORT环境变量被设为了9999

5、USER
USER指令用来指定该镜像启动的容器会以哪个用户运行
```
USER nginx
```
上面的指令表示，基于该镜像启动的容器会以nginx用户的身份来运行。我们还可以指定组名
也可以在docker run命令中通过-u选项来覆盖该指令指定的值，如果不通过USER指令指定用户，默认用户为root

6、VOLUME
VOLUME指令用来向基于镜像创建的容器添加卷。一个卷是可以存在于一个或者多个容器内的特定目录，这个目录可以绕过联合文件系统，并提供一下共享数据或者对数据进行持久化的功能。

- 卷可以在容器间共享和复用
- 一个容器可以不是必须和其他容器共享卷
- 对卷的修改即时生效
- 对卷的修改不会对更新镜像产生影响
- 卷会一直存在直到没有任何容器再使用它

卷功能让我们可以将数据、数据库元数据或者其他内容添加到镜像中而不是将这些内容提交到镜像中，并且允许我们在多个容器间共享这些内容。我们可以利用此功能来测试容器和内部的应用程序代码，管理日志，或者处理容器内部的数据库。
```
VOLUME ["/data"]
```
```
VOLUME ["/data","/var/lib/mysql"]
```
我们也可以在docker run命令行中使用-v参数进行指定。

7、ADD
ADD 指令用来将构建环境下的文件和目录复制到镜像中。
```
 ADD nginx.conf /usr/local/nginx/conf/nginx.conf
```
这里ADD会将构建目录下的nginx.conf文件复制到镜像中/usr/local/nginx/conf/nginx.conf。其中源文件的位置参数可以是一个URL，或者构建上下文的文件名或目录。不能对构建目录或者上下文之外的文件进行ADD操作。
在ADD文件时，docker通过目的地址参数末尾的字符来判断文件源是目录还是文件。如果目标地址是以/结尾，那么docker就认为源位置指向的是一个目录。如果不是，那么docker就认为源位置指向的是文件。如果目标地址不存在的话，docker将会为我们创建这个全路径，包括路径中的任何目录。新创建的文件和目录的模式为0755，并且GID和UID都是0。
ADD指令会使得构建缓存变得无效。如果通过ADD指令向镜像添加一个文件或者目录，那么这将使Dockerfile中的后续指令都不能继续使用之前的构建缓存。
最后ADD在处理归档文件时，会自动进行解压。
```
ADD test.tar.gz  /var/local/test/
```
上面的指令，会将test.tar.gz解压到/var/local/test/目录下面。

以上就是对docker镜像的入门讲解，更多的实战内容，会在后续的博客中体现出来。

