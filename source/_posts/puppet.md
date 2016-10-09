---
title : puppet 安装
date: 2016-8-21
---
## puppet 安装
### 概述
puppet是一个开源的软件自动化配置和部署工具，它使用简单且功能强大，现在很多大型IT公司均在使用puppet对集群中的软件进行管理和部署。

puppet是基于c/s架构的。服务器端保存着所有对客户端服务器的配置代码，在puppet里面叫做manifest. 客户端下载manifest之后，可以根据manifest对服务器进行配置，例如软件包管理，用户管理和文件管理等等。

puppet的工作流程分为以下10个步骤：
1.客户端通过facter收集客户端信息并发送至服务端。
2.连接服务端并请求catalog日志。
3. 请求节点（node）的信息.
4. 从服务器端接收节点（node)的实例
5. 编译代码（包括语法检查等工作）
6.查询是否有exported 虚拟资源
7. 如有，则从数据库接收虚拟资源
8. 接收完整的catalog日志
9. 存储catalog日志到数据库
10.客户端接收完整的catalog日志。

### 安装
**环境说明：**
server：gy-vm02  ip：192,168.1.81
client：gy-vm03   ip：192.168.1.124

配置epel源，直接使用yum进行安装

server端：
yum -y install puppet puppet-server
安装目录默认存为/etc/puppet，该目录下的manifests存放manifest文件。
puppet.conf是主要的配置文件

client端：
yum -y install puppet

安装完成之后，我们在server端启动puppetmaster服务，并打开8140端口。
```
[root@gy-vm02 puppet]# /etc/init.d/puppetmaster start
```

### client端进行认证
首先在在server和client服务器的/etc/hosts里面添加对应hostname和ip。
然后在client上输入一下命令
```
[root@gy-vm03 ~]# puppetd --test --server gy-vm02
info: Creating a new SSL key for gy-vm03
info: Caching certificate for ca
info: Creating a new SSL certificate request for gy-vm03
info: Certificate Request fingerprint (SHA256): B1:E1:35:DB:28:D4:80:4A:F6:DF:CA:26:F1:F8:7C:53:71:D9:DB:B0:4C:C8:B9:EC:D1:77:FF:6F:C8:3C:8E:A7
Exiting; no certificate found and waitforcert is disabled
[root@gy-vm03 ~]#
```
我们可以看到client发了一个认证

###  服务器确认认证
```
[root@gy-vm02 puppet]# puppetca -a --list
  "gy-vm03" (F6:D4:44:EE:D6:02:B4:51:F3:7F:F0:26:EE:89:28:8B)
```
使用puppetca可以查看认证信息，我们可以看到有来至gy-vm03的认证信息
使用下面命令确认信息
```
[root@gy-vm02 puppet]# puppetca -s gy-vm03
notice: Signed certificate request for gy-vm03
notice: Removing file Puppet::SSL::CertificateRequest gy-vm03 at '/var/lib/puppet/ssl/ca/requests/gy-vm03.pe
```
再次查看
```
[root@gy-vm02 puppet]# puppetca -a --list
+ "gy-vm03" (E5:5E:A0:48:AC:BC:14:A4:60:66:E3:11:6C:03:CE:DF)
```
注意前面的+号，说明已经签名

###  总结
puppet的安装，使用 yum安装比较方便，puppet的配置也比较清楚，puppet语法可以
参阅ruby语法，总之来说，puppet安装与配置简单，但是puppet有很多资源模块需要化
时间去阅读






