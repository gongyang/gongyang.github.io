---
title : nginx 平滑升级
date: 2016-5-29
---
## nginx 平滑升级
由于公司需要使用HTTP/2，以及IPv6等新特性，需要在nginx层添加相对应的模块。我们使用的是淘宝开源的tengine，需要下载最新的版本才能支持HTTP/2，[下载地址](http://tengine.taobao.org/download_cn.html)。

#### 编译
升级之前，我们还是先把配置文件copy一份。然后使用nginx -V查看nginx的版本和编译了哪些模块
```
[root@bhqa nginx]# /usr/local/nginx/sbin/nginx -V
Tengine version: Tengine/2.1.0 (nginx/1.6.2)
built by gcc 4.4.7 20120313 (Red Hat 4.4.7-11) (GCC) 
TLS SNI support enabled
configure arguments: --with-http_ssl_module --with-http_gzip_static_module --with-http_stub_status_module
```

解压下载的新版本，进入到解压后的目录，编译安装新版本。这里需要注意的一点是，我们原来在编译的时候，使用的默认安装路径，没有特别指定安装路径。
```
[root@bhqa tengine-2.1.2]# ./configure --with-http_ssl_module --with-http_gzip_static_module --with-http_stub_status_module --with-ipv6 --with-http_v2_module && make && make install
```
#### 升级
首先检查新的nginx二进制文件是否可用
```
[bhuser@bhqa conf]$ sudo /usr/local/nginx/sbin/nginx -t -c /usr/local/nginx/conf/nginx.conf
the configuration file /usr/local/nginx/conf/nginx.conf syntax is ok
configuration file /usr/local/nginx/conf/nginx.conf test is successful
```
接下来我们使用kill -USR2给当前的nginx主进程发送一个信号
```
kill -USR2 `cat /usr/local/nginx/logs/nginx.pid`
```
我们来看下此时的nginx进程
```
[root@bhqa tengine-2.1.2]# ps aux | grep nginx
root      2073  0.0  0.0  71056  2056 ?        Ss   May19   0:00 nginx: master process /usr/local/nginx/sbin/nginx -c /usr/local/nginx/conf/nginx.conf
nobody    2076  0.0  0.0 101300  9396 ?        S    May19   7:07 nginx: worker process                                          
nobody    2077  0.0  0.1 101756 10972 ?        S    May19   7:15 nginx: worker process                                          
root     12857  0.6  0.0  71640  7696 ?        S    15:49   0:00 nginx: master process /usr/local/nginx/sbin/nginx -c /usr/local/nginx/conf/nginx.conf
nobody   12858  0.4  0.3  99288 32396 ?        S    15:49   0:00 nginx: worker process                                          
nobody   12859  0.2  0.3  99288 32396 ?        S    15:49   0:00 nginx: worker process
```
此时的新老进程会同时存在，都会接受并处理请求，并且原来/usr/local/nginx/logs/nginx.pid会改为/usr/local/nginx/logs/nginx.pid.oldbin。

这个时候，我们可以使用kill -WINCH这个信号来让老的worker进程不再接受新的请求并退出
```
kill -WINCH `cat /usr/local/nginx/logs/nginx.pid.oldbin`
```
此时我们再来看下这个时候nginx进程
```
[root@bhqa tengine-2.1.2]# ps aux | grep nginx
root      2073  0.0  0.0  71056  2084 ?        Ss   May19   0:00 nginx: master process /usr/local/nginx/sbin/nginx -c /usr/local/nginx/conf/nginx.conf
root     12857  0.0  0.0  71640  7696 ?        S    15:49   0:00 nginx: master process /usr/local/nginx/sbin/nginx -c /usr/local/nginx/conf/nginx.conf
nobody   12858  0.2  0.3  99288 33780 ?        S    15:49   0:00 nginx: worker process                                          
nobody   12859  0.2  0.3  99288 33820 ?        S    15:49   0:00 nginx: worker process
```
我们发现老的worker都退出了，只剩下一个master进程。此时如果我们由于其他原因，不想升级到新版本了，还是可以回退到旧版本的，具体步骤如下：

- 发送HUP 信号给旧的master process；
- 发送QUIT 信号给新的master process，从容关闭它的worker processes；
- 发送TERM 信号给新的master process, 强迫关闭；
- 如果一些原因新的worker process没有关闭，发送KILL 信号给它；
- 新的master process 关闭后，旧的master process 删除.oldbin后缀，这样就恢复到升级前的状态了。

如果我们不需要回退，那现在就剩下最后一步升级就完成了，使用kill -QUIT信号，让老的master进程也退出
```
kill -QUIT `cat /usr/local/nginx/logs/nginx.pid.oldbin`
```
到此，整个升级我们就完成了，我们来看下当前的nginx版本
```
[root@bhqa tengine-2.1.2]# /usr/local/nginx/sbin/nginx -V
Tengine version: Tengine/2.1.2 (nginx/1.6.2)
built by gcc 4.4.7 20120313 (Red Hat 4.4.7-16) (GCC) 
TLS SNI support enabled
configure arguments: --with-http_ssl_module --with-http_gzip_static_module --with-http_stub_status_module --with-ipv6 --with-http_v2_module
```
版本已是最新的2.1.2，IPv6和HTTP/2相应的模块也编译进去了。
