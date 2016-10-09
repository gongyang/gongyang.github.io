---
title : nginx防止DDOS攻击配置
date: 2016-3-20
---
最近查看服务器nginx日志，看到很多类似DDos攻击的IP被nginx拦截了。
DDos全称分布式拒绝服务(DDoS:Distributed Denial of Service)攻击指借助于客户/服务器技术，将多个计算机联合起来作为攻击平台，对一个或多个目标发动DDoS攻击，从而成倍地提高拒绝服务攻击的威力。
DDOS的特点是分布式，针对带宽和服务攻击，也就是四层流量攻击和七层应用攻击，相应的防御瓶颈四层在带宽，七层的多在架构的吞吐量。对于七层的应用攻击，我们还是可以做一些配置来防御的，例如前端是Nginx，主要使用nginx的http_limit_conn和http_limit_req模块来防御。
ngx_http_limit_conn_module 可以限制单个IP的连接数，ngx_http_limit_req_module 可以限制单个IP每秒请求数，通过限制连接数和请求数能相对有效的防御CC攻击。

###  限制每秒请求数
ngx_http_limit_req_module模块通过漏桶原理来限制单位时间内的请求数，一旦单位时间内请求数超过限制，就会返回503错误。配置需要在两个地方设置：

- nginx.conf的http段内定义触发条件，可以有多个条件
- 在location内定义达到触发条件时nginx所要执行的动作

例如：
```
http {
    limit_req_zone $limit zone=one:10m rate=20r/s; //触发条件，所有访问ip 限制每秒10个请求
    ...
    server {
        ...
        location  ~ \.php$ {
            limit_req zone=one burst=5 nodelay;   //执行的动作,通过zone名字对应
               }
           }
     }
```
**参数说明**
```
$binary_remote_addr  二进制远程地址
zone=one:10m    定义zone名字叫one，并为这个zone分配10M内存，用来存储会话（二进制远程地址），1m内存可以保存16000会话
rate=20r/s;     限制频率为每秒20个请求
burst=5         允许超过频率限制的请求数不多于5个，假设1、2、3、4秒请求为每秒9个，那么第5秒内请求15个是允许的，反之，如果第一秒内请求15个，会将5个请求放到第二秒，第二秒内超过10的请求直接503，类似多秒内平均速率限制。
nodelay         超过的请求不被延迟处理，设置后15个请求在1秒内处理。
```

### 限制IP连接数
ngx_http_limit_conn_module的配置方法和参数与http_limit_req模块很像，参数少，要简单很多
```
http {
    limit_conn_zone $binary_remote_addr zone=addr:10m; //触发条件
    ...
    server {
        ...
        location /download/ {
            limit_conn addr 1;    // 限制同一时间内1个连接，超出的连接返回503
                }
           }
     }
```
### 白名单设置
http_limit_conn和http_limit_req模块限制了单ip单位时间内的并发和请求数，但是如果Nginx前面有lvs或者haproxy之类的负载均衡或者反向代理，nginx获取的都是来自负载均衡的连接或请求，这时不应该限制负载均衡的连接和请求，就需要geo和map模块设置白名单：
```
    geo $boohee {
       default 1;
       127.0.0.1 0;
       192.168.1.0/24 0;
    }
    map $boohee $limit {
       1 $binary_remote_addr;
       0 "";
    }
    limit_req_zone $limit zone=one:10m rate=20r/s;
    limit_conn_zone $limit zone=addr:10m;
```
geo模块定义了一个默认值是1的变量boohee，当在ip在白名单中，变量boohee的值为0，反之为1

如果ip在白名单中，boohee=0，$limit=""，则不受到限制

如果ip不在白名单中，boohee=1，$limit=$binary_remote_addr，受到限制。

