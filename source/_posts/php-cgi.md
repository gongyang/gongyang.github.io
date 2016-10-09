---
title : 使用Spawn-FCGI管理php-cgi
date: 2016-9-25
---
## 使用Spawn-FCGI管理php-cgi
最近在使用nginx的过程中，遇到一个用php来处理动态请求的情况，以前也没接触过php，其中有些概念，还真是有些多，今天特意整理了一下。

我们知道，nginx web服务器只是内容的分发者，早先的web服务器处理静态请求比较多，如果请求/index.html，那么web server会去文件系统中找到这个文件，发送给浏览器，这里分发的是静态数据。但是如果现在请求的是/index.php，根据配置文件，nginx知道这个不是静态文件，需要去找PHP解析器来处理，那么他会把这个请求简单处理后交给PHP解析器。
 
 Nginx会传哪些数据给PHP解析器呢？url要有吧，查询字符串也得有吧，POST数据也要有，HTTP header不能少吧，好的，CGI就是规定要传哪些数据、以什么样的格式传递给后方处理这个请求的协议。

###  CGI
通用网关接口（Common Gateway Interface/CGI）描述了客户端和服务器程序之间传输数据的一种标准，可以让一个客户端，从网页浏览器向执行在网络服务器上的程序请求数据。CGI 独立于任何语言的，CGI 程序可以用任何脚本语言或者是完全独立编程语言实现，只要这个语言可以在这个系统上运行。Unix shell script, Python, Ruby, PHP, perl, Tcl, C/C++, 和 Visual Basic 都可以用来编写 CGI 程序。

当web server收到/index.php这个请求后，会启动对应的CGI程序（php-cgi），这里就是PHP的解析器。接下来PHP解析器会解析php.ini文件，初始化执行环境，然后处理请求，再以规定CGI规定的格式返回处理后的结果，退出进程。web server再把结果返回给浏览器。

CGI使外部程序与Web服务器之间交互成为可能。CGI程式运行在独立的进程中，并对每个Web请求建立一个进程，这种方法非常容易实现，但效率很差，难以扩展。面对大量请求，进程的大量建立和消亡使操作系统性能大大下降。此外，由于地址空间无法共享，也限制了资源重用。这个时候FashCGI应运而生。

### FastCGI
快速通用网关接口（Fast Common Gateway Interface／FastCGI）是通用网关接口（CGI）的改进，描述了客户端和服务器程序之间传输数据的一种标准。FastCGI致力于减少Web服务器与CGI程式之间互动的开销，从而使服务器可以同时处理更多的Web请求。与为每个请求创建一个新的进程不同，FastCGI使用持续的进程来处理一连串的请求。这些进程由FastCGI进程管理器管理，而不是web服务器。

那么Fastcgi是怎么做的呢？
首先，Fastcgi会先启一个master，解析配置文件，初始化执行环境，然后再启动多个worker。当请求过来时，master会传递给一个worker，然后立即可以接受下一个请求。这样就避免了重复的劳动，效率自然是高。而且当worker不够用时，master可以根据配置预先启动几个worker等着；当然空闲worker太多时，也会停掉一些，这样就提高了性能，也节约了资源。这就是fastcgi的对进程的管理。
CGI 就是所谓的短生存期应用程序，FastCGI 就是所谓的长生存期应用程序。FastCGI像是一个常驻(long-live)型的CGI，它可以一直执行着，不会每次都要花费时间去fork一次(这是CGI最为人诟病的fork-and-execute 模式)。

### php-cgi
php-cgi是PHP自带的FastCGI进程，它是php程序的解析器
，本身只能解析请求，返回结果，并不会进程管理，所以才会出现了一些能够调度php-cgi进程的程序，比如PHP-FPM、Spawn-FCGI。


### PHP-FPM、Spawn-FCGI
PHP-FPM、Spawn-FCGI都是守护php-cgi进程的进程管理器。

Spawn-FCGI安装比较简单，使用起来也比较方便，官方[下载地址](http://www.lighttpd.net/download/)。
安装完成之后使用如下命令启动
```
 /usr/local/bin/spawn-fcgi -a 127.0.0.1 -p 9000 -C 5 -u www-data -g www-data -f /usr/bin/php-CGI
```
参数含义如下：
```
   -f 指定调用FastCGI的进程的执行程序位置，根据系统上所装的PHP的情况具体设置
　　-a 绑定到地址addr
　　-p 绑定到端口port
　　-s 绑定到unix socket的路径path
　　-C 指定产生的FastCGI的进程数，默认为5(仅用于PHP)
　　-P 指定产生的进程的PID文件路径
　　-u和-g FastCGI使用什么身份(-u 用户 -g 用户组)运行
```

### 总结
CGI、FastCGI是一种协议，PHP-FPM和Spawn-FCGI是实现了这个协议的一种软件。php-cgi只是解释PHP脚本的程序而已。
