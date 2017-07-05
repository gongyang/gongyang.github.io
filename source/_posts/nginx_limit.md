---
title : nginx limit_conn  limit_req模块讲解
date: 2016-10-16
---
## nginx limit_conn  limit_req模块讲解
nginx针对于限流，提供了两个模块：连接数限流模块 ngx_http_limit_conn_module 和漏桶算法实现的请求限流模块 ngx_http_limit_req_module 。

limit_conn 用来对某个 KEY 对应的总的网络连接数进行限流，可以按照如 IP 、域名维度进行限流。 limit_req 用来对某个 KEY 对应的请求的平均速率进行限流，并有两种用法：平滑模式（ delay ）和允许突发模式 (nodelay) 。

### ngx_http_limit_conn_module
limit_conn 是对某个 KEY 对应的总的网络连接数进行限流。可以按照 IP 来限制 IP 维度的总连接数，或者按照服务域名来限制某个域名的总连接数。但是记住不是每一个请求连接都会被计数器统计，只有那些被 Nginx 处理的且已经读取了整个请求头的请求连接才会被计数器统计。

**配置示例**
```
http {  
    log_format mylogformat [$time_local] [$msec] '$status';
    limit_req_zone $binary_remote_addr zone=ggyy:10m rate=2r/s;
    server {
    listen 80;
    server_name abc.gongyang.com;
    root /var/www/html;     
    location /a.txt {
    limit_req zone=ggyy burst=3 nodelay;
    access_log logs/abc.log mylogformat;
    }
}
```
**配置说明**
limit_conn ：要配置存放 KEY 和计数器的共享内存区域和指定 KEY 的最大连接数；此处指定的最大连接数是 1 ，表示 Nginx 最多同时并发处理 1 个连接；
limit_conn_zone ：用来配置限流 KEY 、及存放 KEY 对应信息的共享内存区域大小；此处的 KEY 是“ \$binary_remote_addr ”其表示 IP 地址，也可以使用如 \$server_name 作为 KEY 来限制域名级别的最大连接数；
limit_conn_status ：配置被限流后返回的状态码，默认返回 503 ；
limit_conn_log_level ：配置记录被限流后的日志级别，默认 error 级别。


**limit_conn 的主要执行过程如下所示：**
1 、请求进入后首先判断当前 limit_conn_zone 中相应 KEY 的连接数是否超出了配置的最大连接数；
2.1 、如果超过了配置的最大大小，则被限流，返回 limit_conn_status 定义的错误状态码；
2.2 、否则相应 KEY 的连接数加 1 ，并注册请求处理完成的回调函数；
3 、进行请求处理；
4 、在结束请求阶段会调用注册的回调函数对相应 KEY 的连接数减 1 。
limt_conn 可以限流某个 KEY 的总并发 / 请求数， KEY 可以根据需要变化。

### ngx_http_limit_rep_module
limit_req 是令牌桶算法实现，用于对指定 KEY 对应的请求进行限流，比如按照 IP 维度限制请求速率。

**配置示例**
```
http {
    include       mime.types;
    default_type  application/octet-stream;
    sendfile        on;
    keepalive_timeout  65;
  
    log_format mylogformat [$time_local] [$msec] '$status';
    limit_req_zone $binary_remote_addr zone=ggyy:10m rate=2r/s;
    server {
    listen 80;
    server_name abc.gongyang.com;
    root /var/www/html;     

    location /a.txt {
    limit_req zone=ggyy burst=3 nodelay;
    access_log logs/abc.log mylogformat;
    }
}
```
**配置说明**
limit_req ：配置限流区域、桶容量（突发容量，默认 0 ）、是否延迟模式（默认延迟）；
limit_req_zone ：配置限流 KEY 、及存放 KEY 对应信息的共享内存区域大小、固定请求速率；此处指定的 KEY 是“\$binary_remote_addr ”表示 IP 地址；固定请求速率使用 rate 参数配置，支持 10r/s 和 60r/m ，即每秒 10 个请求和每分钟 60 个请求，不过最终都会转换为每秒的固定请求速率（ 10r/s 为每 100 毫秒处理一个请求； 60r/m ，即每 1000 毫秒处理一个请求）。
limit_conn_status ：配置被限流后返回的状态码，默认返回 503 ；
limit_conn_log_level ：配置记录被限流后的日志级别，默认 error 级别。

**limit_req 的主要执行过程如下所示：**
1 、请求进入后首先判断最后一次请求时间相对于当前时间（第一次是 0 ）是否需要限流，如果需要限流则执行步骤 2 ，否则执行步骤 3 ；
2.1 、如果没有配置桶容量（ burst ），则桶容量为 0 ；按照固定速率处理请求；如果请求被限流，则直接返回相应的错误码（默认 503 ）；
2.2 、如果配置了桶容量（ burst>0 ）且延迟模式 ( 没有配置 nodelay) ；如果桶满了，则新进入的请求被限流；如果没有满则请求会以固定平均速率被处理（按照固定速率并根据需要延迟处理请求，延迟使用休眠实现）；
2.3 、如果配置了桶容量（ burst>0 ）且非延迟模式（配置了 nodelay ）；不会按照固定速率处理请求，而是允许突发处理请求；如果桶满了，则请求被限流，直接返回相应的错误码；
3 、如果没有被限流，则正常处理请求；
4 、 Nginx 会在相应时机进行选择一些（ 3 个节点）限流 KEY 进行过期处理，进行内存回收。
