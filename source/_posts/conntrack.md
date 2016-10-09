---
title : 系统内核参数--nf_conntrack
date: 2016-9-4
---
## 系统内核参数--nf_conntrack
### 基本概念
nf_conntrack 是 iptables 防火墙模块，它是用于跟踪一个连接的状态的模块。

由于nf_conntrack 工作在3层，支持IPv4和IPv6，而ip_conntrack只支持IPv4，因此nf_conntrack模块在Linux的2.6.15内核中被引入，而ip_conntrack在Linux的2.6.22内核被移除（centos6.x版本），因此不同版本的系统，配置文件也就不尽相同了。目前大多的ip_conntrack_*已被 nf_conntrack_* 取代，很多ip_conntrack_*仅仅是个软链接，原先的ip_conntrack配置目录/proc/sys/net/ipv4/netfilter/ 仍然存在，但是新的nf_conntrack在/proc/sys/net/netfilter/中，这样做是为了能够向下的兼容。

ip_conntrack 模块会记录每一个tcp数据包的estiablished,timewait,syn_recv等状态。最常见的两个使用场景是 iptables的nat和state模块。

iptables 的 nat 通过规则来修改目的、源地址，但光修改地址不行，我们还需要能让回来的包能路由到最初的来源主机。这就需要借助 nf_conntrack 来找到原来那个连接的记录才行。
而 state 模块则是直接使用 nf_conntrack 里记录的连接的状态来匹配用户定义的相关规则。

### 参数设定
kernel 用 ip_conntrack 模块来记录 iptables 网络包的状态，并把每条记录保存到 table 里（这个 table 在内存里，可以通过/proc/net/ip_conntrack 查看当前已经记录的总数），如果网络状况繁忙，比如高连接，高并发连接等会导致逐步占用这个 table 可用空间，一般这个 table 很大不容易占满并且可以自己清理，table 的记录会一直呆在 table 里占用空间直到源 IP 发一个 RST 包，但是如果出现被攻击、错误的网络配置、有问题的路由/路由器、有问题的网卡等情况的时候，就会导致源 IP 发的这个 RST 包收不到，这样就积累在 table 里，越积累越多直到占满，满了以后 iptables 就会丢包，出现外部无法连接服务器的情况。内核会报如下错误信息：kernel: ip_conntrack: table full, dropping packet.

系统当前连接可以查看/proc/net/nf_conntrack
```
[root@gy-vm02 ~]# cat /proc/net/nf_conntrack
ipv4     2 tcp      6 299 ESTABLISHED src=192.168.168.1 dst=192.168.168.11 sport=53635 dport=22 src=192.168.168.11 dst=192.168.168.1 sport=22 dport=53635 [ASSURED] mark=0 secmark=0 use=3
ipv4     2 udp      17 28 src=192.168.1.4 dst=192.168.1.255 sport=137 dport=137 [UNREPLIED] src=192.168.1.255 dst=192.168.1.4 sport=137 dport=137 mark=0 secmark=0 use=2
ipv4     2 udp      17 28 src=192.168.1.1 dst=255.255.255.255 sport=38632 dport=7423 [UNREPLIED] src=255.255.255.255 dst=192.168.1.1 sport=7423 dport=38632 mark=0 secmark=0 use=2
```

系统当前参数值可以通过syscrl -a来查看
```
[root@gy-vm02 ~]# sysctl -a | grep nf_conntrack
net.netfilter.nf_conntrack_generic_timeout = 600
net.netfilter.nf_conntrack_tcp_timeout_syn_sent = 120
net.netfilter.nf_conntrack_tcp_timeout_syn_recv = 60
net.netfilter.nf_conntrack_tcp_timeout_established = 4000
net.netfilter.nf_conntrack_tcp_timeout_fin_wait = 120
net.netfilter.nf_conntrack_tcp_timeout_close_wait = 60
net.netfilter.nf_conntrack_tcp_timeout_last_ack = 30
net.netfilter.nf_conntrack_tcp_timeout_time_wait = 120
net.netfilter.nf_conntrack_tcp_timeout_close = 10
net.netfilter.nf_conntrack_tcp_timeout_max_retrans = 300
net.netfilter.nf_conntrack_tcp_timeout_unacknowledged = 300
net.netfilter.nf_conntrack_tcp_loose = 1
net.netfilter.nf_conntrack_tcp_be_liberal = 0
net.netfilter.nf_conntrack_tcp_max_retrans = 3
net.netfilter.nf_conntrack_udp_timeout = 30
net.netfilter.nf_conntrack_udp_timeout_stream = 180
net.netfilter.nf_conntrack_icmpv6_timeout = 30
net.netfilter.nf_conntrack_icmp_timeout = 30
net.netfilter.nf_conntrack_acct = 0
net.netfilter.nf_conntrack_events = 1
net.netfilter.nf_conntrack_events_retry_timeout = 15
net.netfilter.nf_conntrack_max = 200000
net.netfilter.nf_conntrack_count = 4
net.netfilter.nf_conntrack_buckets = 16384
net.netfilter.nf_conntrack_checksum = 1
net.netfilter.nf_conntrack_log_invalid = 0
net.netfilter.nf_conntrack_expect_max = 256
net.ipv6.nf_conntrack_frag6_timeout = 60
net.ipv6.nf_conntrack_frag6_low_thresh = 3145728
net.ipv6.nf_conntrack_frag6_high_thresh = 4194304
net.nf_conntrack_max = 200000
```
以上的内容实际在/proc/sys/net/netfilter都有对应的文件配置。

对于普通的服务器来说，默认参数不会影响正常使用，但如果是对外web服务器，我们通常要修改这几个值。
**直接设置：**
```
[root@gy-vm02 ~]# sysctl -w net.netfilter.nf_conntrack_max=100000
net.netfilter.nf_conntrack_max = 100000
[root@gy-vm02 ~]# sysctl -w net.netfilter.nf_conntrack_tcp_timeout_established=3600
net.netfilter.nf_conntrack_tcp_timeout_established = 3600
```
或者
```
[root@gy-vm02 ~]# echo "3600" > /proc/sys/net/netfilter/nf_conntrack_tcp_timeout_established 
[root@gy-vm02 ~]# echo "100000" > /proc/sys/net/netfilter/nf_conntrack_max 
```
**修改配置文件：**
打开配置文件sysctl.conf ```vim /etc/sysctl.conf``` 加入上面需要修改的信息
```
net.netfilter.nf_conntrack_max = 100000
net.netfilter.nf_conntrack_tcp_timeout_established = 3600
```
然后使用```sysctl -p``` 生效。


