---
title : Nginx 配置文件详解
date: 2016-10-23
---
## Nginx 配置文件详解
```
main
events {
  .... } http {   upstream myproject_svr {     .....   } 
  server { 
    location { 
      .... 
    }
  }    
  server { 
    location { 
       .... 
    } 
  }
    .... 
}
```
nginx配置文件主要分为六个区域：
main (全局设置)、 events (nginx工作模式)、 http (http设置)、 server (主机设置)、 location (URL匹配)、 upstream (负载均衡服务器设置)。
###main全局设置模块
这一块区域主要设置一些全局的属性，比如用户组、进程数、错误日志等。我们来看一下具体配置：
```
user www www;  
worker_processes 8;  
error_log /data1/logs/apps/nginx/nginx_error.log  crit;  
pid /usr/local/run/nginx.pid;  
worker_rlimit_nofile 131070;
```
user 指定nginx worker进程的运行用户以及用户组，默认由nobody账号运行。
worker_processes 指定nginx要开启的子进程数。每个nginx进程平均耗费10M~12M内存。根据经验，一般指定1个进程就足够了，如果是多核CPU，建议指定和CPU的数量一样的进程数即可。这里写2，那么就会开启2个子进程。
error_log 定义全局错误日志文件。日志输出级别有debug、info、notice、warn、error、crit可供选择，其中，debug输出日志最为最详细，而crit输出日志最少。
pid 指定进程id的存储文件位置。
worker_rlimit_nofile 指定一个nginx进程可以打开的最多文件描述符数目。

### events模块

events模块用于指定nginx的工作模式和连接数上限，一般是这样：
```
events {  
    use epoll;
    worker_connections 131070;
}
```
use 指定nginx的工作模式。nginx支持的工作模式有 select 、 poll 、 kqueue、 epoll 、 rtsig 和 /dev/poll 。其中 select 和 poll 都是标准的工作模式， kqueue 和 epoll 是高效的工作模式，不同的是 epoll 用在Linux平台上，而 kqueue 用在BSD系统中，因为Mac基于BSD,所以Mac也得用这个模式，对于Linux系统，epoll工作模式是首选。
worker_connections 定义nginx每个进程的最大连接数，即接收前端的最大请求数，默认是1024。最大客户端连接数由worker_processe和worker_connections决定，即max_clients=worker_processes*worker_connections。
### http模块
http模块可以说是最核心的模块了，它负责HTTP服务器相关属性的配置，它里面的server和upstream子模块，至关重要，后面会仔细说。
```
http {  
    include       mime.types;
    default_type  application/octet-stream;
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';
    access_log  /usr/local/var/log/nginx/access.log  main;
    sendfile        on;
    tcp_nopush      on;
    tcp_nodelay     on;
    keepalive_timeout  10;
    gzip  on;
    upstream myproject {
        .....
    }
    server {
        ....
    }
}
```
下面详细介绍下这段代码中每个配置选项的含义。 include 设定文件的mime类型，类型在配置文件目录下的mime.types文件定义，来告诉nginx来识别文件类型。
default_type 设定了默认的类型为二进制流，也就是当文件类型未定义时使用这种方式了。
log_format 设置日志的格式，这里的 main 用于access_log来纪录这种类型。
main的类型日志如下：也可以增删部分参数。
```
127.0.0.1 - - [21/Apr/2015:18:09:54 +0800] "GET /index.php HTTP/1.1" 200 87151 "-" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_10_2) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/41.0.2272.76 Safari/537.36"
```
access_log 纪录每次的访问日志的文件地址，后面的main是日志的格式样式，对应于log_format的main。
sendfile 开启高效文件传输模式。将 tcp_nopush 和 tcp_nodelay 两个指令设置为on用于防止网络阻塞。
keepalive_timeout 设置客户端连接保持活动的超时时间。在超过这个时间之后，服务器会关闭该连接。
gzip 设置是否开启gzip压缩。
upstream 负载均衡设置。
server 虚拟主机设置。
还有很多各种配置，以后等用到来再说。

### server模块

sever模块是http的子模块，它用来定一个虚拟主机，我们先讲最基本的配置，这些在后面再讲。
我们看一下一个简单的server 是如何做的？
```
server {  
        listen       8080;
        server_name  fanxing.kugou.com www.fanxing.com fanxing.com;
        # 全局定义，如果都是这一个目录，这样定义最简单。
        root   /data1/htdocs/fanxing.com;
        index  index.php index.html index.htm; 
        charset utf-8;
        access_log  usr/local/var/log/host.access.log  main;
        error_log  usr/local/var/log/host.error.log  error;
        ....
}
```
server 标志定义虚拟主机开始。
listen 用于指定虚拟主机的服务端口。
server_name 用来指定IP地址或者域名，多个域名之间用空格分开。
root 用于配置项目的根目录。
index 全局定义访问的默认首页地址。
charset 用于设置网页的默认编码格式。
access_log 用来指定此虚拟主机的访问日志存放路径，最后的main用于指定访问日志的输出格式。
error_log 用来指定此虚拟主机的错误日志存放路径，最后的error用于指定错误日志的输出格式。

### location模块

location 模块是nginx中用的最多的，也是最重要的模块了。什么负载均衡啊、反向代理啊、虚拟域名啊都与它相关。
location 根据它字面意思就知道是来定位的，定位URL，解析URL，所以，它也提供了强大的正则匹配功能，也支持条件判断匹配，用户可以通过location指令实现Nginx对动、静态网页进行过滤处理。
location 支持路径匹配也支持正则匹配，我们可以通过 location 强大的匹配功能来定位我们的web服务。我们在这里只介绍一下基本用法：

路径匹配 我们先来看这个，设定默认首页和虚拟机目录。
```
location / {  
    root   html;    
    index  index.html index.htm;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header Host $http_host;
    proxy_set_header X-NginX-Proxy true;
    proxy_pass http://127.0.0.1:3000/;
    proxy_redirect off;
}
```
location / 表示匹配访问根目录。
root 指令用于指定访问根目录时，虚拟主机的web目录，这个目录可以是相对路径（相对路径是相对于nginx的安装目录）。也可以是绝对路径。
index 用于设定我们只输入域名后访问的默认首页地址，有个先后顺序：index.html index.htm，如果没有开启目录浏览权限，又找不到这些默认首页，就会报403错误。
proxy_set_header 设置http请求头。
proxy_pass 指定匹配访问的服务；
正则匹配 location 还有一种方式就是正则匹配，开启正则匹配这样： location ~ ，也就是后面加个 ~ 。

看一看我们开发服的php配置：
```
location ~.*\.(php|php5)?$ {  
    fastcgi_pass 127.0.0.1:9000;
    fastcgi_index index.php;
    include fcgi.conf;
}
```
这个正则的意思就是匹配以php或者php5结尾的访问路径。里面的fastcgi_pass是用来表示虚拟主机php-fpm服务地址。
location 还有很多其他用法，想要详细的童鞋可以继续阅读我的下一篇博文：
### upstream模块
upstream 模块负责 负载均衡 模块，通过简单的调度算法来实现客户端IP到后端服务器的负载均衡。
我们先看一份配置：
```
upstream myproject_svr {  
    weight;
    server 10.1.2.1:80 weight=8 max_fails=5 fail_timeout=10s;
    server 10.1.2.2:80 weight=8 max_fails=5 fail_timeout=10s;
    server 10.1.2.3:80 weight=8 max_fails=5 fail_timeout=10s;down;
    check interval=3000 rise=2 fall=5 timeout=3000 type=tcp;
    keepalive 16;
}
```
在上面的例子中，通过 upstream 指令指定了一个负载均衡器的名称myproject_svr 。这个名称可以任意指定，在后面需要的地方直接调用即可。
weight 这是其中的一种负载均衡调度算法，nginx的负载均衡模块目前支持4种调度算法:
weight 轮询（默认）。每个请求按时间顺序逐一分配到不同的后端服务器，如果后端某台服务器宕机，故障系统被自动剔除，使用户访问不受影响。server中的 weight指定轮询权值， weight 值越大，分配到的访问机率越高，主要用于后端每个服务器性能不均的情况下。
ip_hash 每个请求按访问IP的hash结果分配，这样来自同一个IP的访客固定访问一个后端服务器，有效解决了动态网页存在的session共享问题。
fair 比上面两个更加智能的负载均衡算法。此种算法可以依据页面大小和加载时间长短智能地进行负载均衡，也就是根据后端服务器的响应时间来分配请求，响应时间短的优先分配。nginx本身是不支持fair的，如果需要使用这种调度算法，必须下载nginx的 upstream_fair 模块。
url_hash 按访问url的hash结果来分配请求，使每个url定向到同一个后端服务器，可以进一步提高后端缓存服务器的效率。nginx本身是不支持 url_hash 的，如果需要使用这种调度算法，必须安装nginx的hash软件包。
紧接着就是各种服务器配置了。在 upstream 模块中，可以通过 server 指令指定后端服务器的IP地址和端口，同时还可以设定每个后端服务器在负载均衡调度中的状态。 常用的状态 有：
down 表示当前的server暂时不参与负载均衡。
backup 预留的备份机器。当其他所有的非backup机器出现故障或者忙的时候，才会请求backup机器，因此这台机器的压力最轻。
max_fails 允许请求失败的次数，默认为1。
fail_timeout 在经历了 max_fails 次失败后，暂停服务的时间。 max_fails 可以和 fail_timeout 一起使用。
注意当负载调度算法为 ip_hash 时，后端服务器在负载均衡调度中的状态不能是weight和backup。
check 指令可以打开后端服务器的健康检查功能。指令后面的参数意义是：
interval ：向后端发送的健康检查包的间隔。
fall(fall_count) : 如果连续失败次数达到fall
count，服务器就被认为是down。
rise(rise_count) count，服务器就被认为是up。
timeout : 后端健康请求的超时时间。
keepalive 表示活动连接最大数量。
至此， nginx.conf 基本的配置就差不多分析完了。

