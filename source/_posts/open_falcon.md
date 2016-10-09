---
title : 小米开源监控open-falcon
date: 2016-6-19
---
##  小米开源监控open-falcon
最近一段时间都在研究小米开源的监控项目open-falcon，为什么要研究它，当然是要用到它才回去学习它。做为一款国内开源的监控项目，在某些功能细节还有待完善，但还是有其自己特点所在。

具体安装我这里就不啰嗦了，详细请参考[官方文档](http://book.open-falcon.org/zh/index.html)。

### 体系架构
了解一款软件，我们先从它的体系架构入手，下面这个就是open-falcon的体积结构图
![](http://image20.it168.com/201510_0x0/2342/78d0080e1afb0674.png)

- falcon-agent：agent用于采集机器负载监控指标，自动收集200多项指标，每隔60秒push到tranfer组件，只要安装了agent，就会自动采集数据，主动上报，不需要在server端做任何配置。
- transfer：transfer是数据转发服务，它接收agent上传的数据，然后通过一次性哈希对数据分片，然后将分片后的数据push到judge和graph组件。
- judge：judge会对transfer上传上来的数据都会进行告警判断，如果数据量比较大，一台机器肯定抗不住，所以在transfer层就对数据进行了分片，后端配置多个judge，每个judge处理一小部分数据。
- alarm ：judge产生的报警event会存放到redis队列，alarm会去redis队列里面拿数据，并选择告警方式。alarm并不自己发送邮件短信，是由sender模块来完成的。
- graph ：graph接收transfer上传上来的数据，做为绘图数据的组件，参考rrdtool的理念，在数据每次存入的时候，会自动进行采样、归档。
- query ：query组件，提供统一的绘图数据查询入口。query组件接收查询请求，根据一致性哈希算法去相应的graph实例查询不同metric的数据，然后汇总拿到的数据，最后统一返回给用户。
- dashboard ：dashboard是面向用户的查询界面。在这里，用户可以看到push到graph中的所有数据，并查看其趋势图。
- portal ：这个是用来配置报警策略的，数据存在mysql里面
- Heartbeat Server： 心跳服务器，公司所有agent都会连到HBS，每分钟发一次心跳请求。


以上这些，就是open-falcon的主要模块，了解了这些模块，部署起来就很方便了。

**重点**
我们知道portal用来配置监控策略的，但是portal数据库中有一个host表，用来记录公司所有机器信息，这些信息都是HBS在维护。agent在连接HBS的同时，会把hostname、ip、agent version、plugin version等信息告诉HBS，HBS负责更新host表。还有一点就是，agent虽然能收集200多项指标，但是类似与端口进程监控，agent不可能全部上传，这也不太现实。open-falcon换了另外一种思路，只对portal配置了的端口进行数据上报。那agent如何知道portal里面配置了哪些监控端口了。还是HBS，HBS会去读portal数据库，然后告诉agent。

除了falcon-agent采集的数据可以push到监控系统，我们还可以自己自定义的一些数据指标，也可以push到open-falcon中。agent提供了一个http接口/v1/push用于接收用户手工push的一些数据，然后通过长连接迅速转发给Transfer。

**特点** 
1、自动发现，支持falcon-agent、snmp、支持用户主动push、用户自定义插件支持
2、支持每个周期上亿次的数据采集、告警判定、历史数据存储和查询
3、高效的portal、支持策略模板、模板继承和覆盖、多种告警方式、支持callback调用
4、单机支撑200万metric的上报、归档、存储
5、采用rrdtool的数据归档策略，秒级返回上百个metric一年的历史数据
6、多维度的数据展示，用户自定义Screen
7、通过各种插件目前支持Linux、Windows、Mysql、Redis、Memache、RabbitMQ和交换机监控。

**缺点**
由于发布时间较短，很多基础的服务监控插件(如Tomcat、apache等)还不支持，很多功能还在不断完善中，另外由于缺少专门的支持，虽然有开放社区，但是解决问题的效率相对较低。

