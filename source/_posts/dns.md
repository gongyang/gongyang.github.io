---
title: DNS -- 查询讲解
date: 2016-5-22
---
### 前言
随着IPv4地址的枯竭，IPv6已经开始践行推广起来，可是IPv6是128bits，如果转换成十进制那可得有16个，使用起来会很麻烦，所以以后会越来越普遍使用主机名来进行通讯，这时主机名自动解析为IP就很重要了。DNS服务器，也就越来越重要了。

###  第一代“DNS”
最开始的因特网，是没有DNS这个概念的，人们使用TCP/IP协议进行通讯，其中IP为第四版的IPv4，由32个位元组成，转换成十进制的数字就是：100.1.9.88这样子的格式。数据包就是通过这样的IP进行传送数据。
可是人脑对于数字的记忆比较费力，尤其是IP越来越多的时候，可是没有IP又不能上internet，那怎么办了？早期的人们想到了一个办法，那就是把IP和主机名一一对应起来，记录到一个特定的档案中，如此一来，我们就可以通过主机名来获得主机IP了，记主机名比记IP要容易的多。/etc/hosts就是那个特定的档案。

这样的方式一开始还比较好用，可是随着internet的推广，需要记录的主机名越来越多，而且主机名和IP这种对应无法自动同步到所有的计算机内，并且要将主机名加入到档案，只能想INTERNIC注册，很不利于同步。

![](http://vbird.dic.ksu.edu.tw/linux_server/0350dns_files/hosts.gif)
早期透过单一档案进行网络联机的示意图

### 分布式、阶层式主机名管理架构： DNS 系统
随着internet的大面积使用，单一的档案/etc/hosts的联网问题就遇到了瓶颈，就会发生上述问题。为了解决这个日益严重的问题，柏克利大学研发出一套阶层式管理主机名对应IP的系统，我们称之为 Berkeley Internet Name Domain,简称BIND。这也是目前全世界使用最广泛的领域名系统（Domain Name System, DNS），通过DNS，我们不需要知道主机IP，只要知道主机名称就可以连上该主机了。

DNS利用类似树状目录的架构，将主机名的管理分配在不同层级的DNS服务器当中，经由分层管理，所以每一部DNS服务器记录的信息就不会很多，而且若有IP变动时也很容易修改。因为只要你申请到了主机名解析的授权，那么在你自己的DNS服务器中就能修改，不需要通过上层的ISP来维护。

在分析原理之前，我们先了解几个概念。

#### FQDN
FQDN（Fully Qualified Domain Name）完整主机名，是由主机名和领域名（hostname and domain name）组成。那什么是主机名和领域名了？
举个例子：
长沙有个电话号码4802802，上海也有个电话号码4802802，如果我在长沙本地拨打4802802就会直接拨通到长沙，如果我想要打到上海的4802802，就得在前面加上区号021，才能拨打到上海。这个时候区号021就是对应的领域名，而电话号码就是对应的主机名了。他们组合起来，就是完整主机名，FQDN。
薄荷的主页是www.boohee.com，其中www就是主机名，boohee.com就是领域名，百度的主页主机名也是www，但是他的领域名是baidu.com，这样就能区分同名的主机了。
前面讲到DNS是以树状目录分阶层的方式来处理主机名，那么这个树状目录中主机名和领域名怎么区分了，我们来看下面图片：
![](http://vbird.dic.ksu.edu.tw/linux_server/0350dns_files/www_ksu.gif)
对于第二层来说，第一层的.tw就是领域名（domain name），而gov、edu、com就是主机名（hostname），而对于第三层来说，edu.tw就变成了domain name，而ntu、ksu就成了hostname。

#### DNS的阶层架构与TLD
![](http://www.goywzl.com/images/server/dns.jpg)
在整个DNS系统的最上方一定是 . （root根），最早以前他底下管理的就只有com、edu、gov、mil、org、net这种特殊领域以及以国家为分类的第二层主机名。这两者称为 Top Level Domains (TLDs)

- 一般最上层领域名 (Generic TLDs, gTLD)：例如 .com, .org, .gov 等等
- 国码最上层领域名 (Country code TLDs, ccTLD)：例如 .tw, .uk, .jp, .cn 等

最早的root仅管理六大领域名，分别如下：

|名称|代表意义| 
|--------|-----|
|com|公司、行号、企业| 
|org|组织、机构| 
|edu|教育单位|
|gov|政府单位|
|net|网络、通讯|
|mil|军事单位|

但是因特网发展的速度太快了，因此后来除了上述的六大类别之外，还有诸如.asia .info .jobs等领域名的开放。此外，为了让某些国家也能够有自己的最上层领域名，因此就有所谓的ccTLD了。每个国家有了最上层ccTLD，所以如果有domain name的需求，则只要向自己的国家申请即可，不需要到root那里去申请。

#### 授权与分层负责
既然TLD这么好，那么是否我们可以随便设置TLD了。当然不可以，我们需要向上层的ISP申请领域名的授权才行。例如中国的最上层是领域名是.cn开头，管理这个领域的服务器必须向root(.)注册领域名查询授权才行。
当得到授权之后，我们就可以自己管理辖下的主机名或子领域，例如.cn下面也可以划分root管理的那六大类。并且，由于DNS是所谓的阶层式的管理，.cn只记录它下面那一层的主机信息而已。至于sina.com.cn，zol.com.cn是由com.cn这台机器来管理，总结来说**每个上一层的DNS服务器所记录的信息，其实只有其下一层的主机名而已**，至于再下一层，就直接授权给下层的某个主机来管理。
这样做的好处就是，每台机器只有下一层的hostname对应的IP而已，所以减少了管理上的困扰！而下层 Client 端如果有问题，只要询问上一层的 DNS server 即可！不需要跨越上层，除错上面也会比较简单！

####  DNS查询流程
![](http://dudns.baidu.com/static/uploads/images/dns-query_20151207015631_954.png)

```
1、用户发出查询请求，首先会去找local dns，也就是我们本地设置的dns服务器，如果在local dns服务器里面找到www.baiduc.com的ip记录就直接返回给用户，然后用户拿着ip去访问百度服务器，如果没找到，就会执行我们上图中的第二步；
2、如果local dns中没有找到www.baiduc.com对应的ip，local dns服务器首先会向root（根服务器）发出查询请求，但是root服务器只记录了com服务器的ip，local dns拿到com服务器的ip，就会有上图的④查询
3、当查询到com服务器，com服务器只记录了baidu.com服务器的ip，接下来local dns又去询问baidu.com服务器，也就是第⑥步
4、当找到baidu.com服务器的时候，这台dns服务器发现自己下面正好记录了www这一台主机，并返回ip给local dns，然后local dns再返回给用户，最终用户得到了www.baidu.com服务器的ip。

在linux或mac系统下面，我们可以使用dig命令，追踪这个过程。
[ggyy@gy-vm02 one]$ dig +trace www.baidu.com

; <<>> DiG 9.8.2rc1-RedHat-9.8.2-0.30.rc1.el6 <<>> +trace www.baidu.com
;; global options: +cmd
.			323984	IN	NS	c.root-servers.net.
.			323984	IN	NS	j.root-servers.net.
.			323984	IN	NS	d.root-servers.net.
.			323984	IN	NS	b.root-servers.net.
.			323984	IN	NS	h.root-servers.net.
.			323984	IN	NS	g.root-servers.net.
.			323984	IN	NS	l.root-servers.net.
.			323984	IN	NS	a.root-servers.net.
.			323984	IN	NS	f.root-servers.net.
.			323984	IN	NS	k.root-servers.net.
.			323984	IN	NS	m.root-servers.net.
.			323984	IN	NS	e.root-servers.net.
.			323984	IN	NS	i.root-servers.net.
;; Received 228 bytes from 210.22.84.3#53(210.22.84.3) in 77 ms

com.			172800	IN	NS	i.gtld-servers.net.
com.			172800	IN	NS	c.gtld-servers.net.
com.			172800	IN	NS	j.gtld-servers.net.
com.			172800	IN	NS	g.gtld-servers.net.
com.			172800	IN	NS	d.gtld-servers.net.
com.			172800	IN	NS	f.gtld-servers.net.
com.			172800	IN	NS	k.gtld-servers.net.
com.			172800	IN	NS	e.gtld-servers.net.
com.			172800	IN	NS	l.gtld-servers.net.
com.			172800	IN	NS	m.gtld-servers.net.
com.			172800	IN	NS	b.gtld-servers.net.
com.			172800	IN	NS	a.gtld-servers.net.
com.			172800	IN	NS	h.gtld-servers.net.
;; Received 491 bytes from 192.112.36.4#53(192.112.36.4) in 371 ms

baidu.com.		172800	IN	NS	dns.baidu.com.
baidu.com.		172800	IN	NS	ns2.baidu.com.
baidu.com.		172800	IN	NS	ns3.baidu.com.
baidu.com.		172800	IN	NS	ns4.baidu.com.
baidu.com.		172800	IN	NS	ns7.baidu.com.
;; Received 201 bytes from 192.41.162.30#53(192.41.162.30) in 272 ms

www.baidu.com.		1200	IN	CNAME	www.a.shifen.com.
a.shifen.com.		1200	IN	NS	ns3.a.shifen.com.
a.shifen.com.		1200	IN	NS	ns1.a.shifen.com.
a.shifen.com.		1200	IN	NS	ns4.a.shifen.com.
a.shifen.com.		1200	IN	NS	ns2.a.shifen.com.
a.shifen.com.		1200	IN	NS	ns5.a.shifen.com.
;; Received 228 bytes from 220.181.37.10#53(220.181.37.10) in 34 ms
```

#### DNS 使用的 port number
DNS使用的53这个端口，我们可以在linux下面的/etc/services收索一下。
```
[ggyy@gy-vm02 ~]$ grep domain /etc/services 
67:domain          53/tcp                          # name-domain server
68:domain          53/udp
10025:domaintime      9909/tcp                # domaintime
10026:domaintime      9909/udp                # domaintime
```
通常 DNS 查询的时候，是以 udp 这个较快速的数据传输协议来查询的， 但是万一没有办法查询到完整的信息时，就会再次的以 tcp 这个协定来重新查询的！所以启动 DNS 的 daemon (就是 named 啦) 时，会同时启动 tcp 及 udp 的 port 53 喔！所以，记得防火墙也要同时放行 tcp, udp port 53。


