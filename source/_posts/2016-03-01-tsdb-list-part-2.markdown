---
layout: post
title: "时序列数据库武斗大会之TSDB名录 Part 2"
date: 2016-03-01 20:56:37 +0800
comments: true
categories: 
styles: [data-table]
---


在前面[《时序列数据库武斗大会之TSDB名录 Part 1》](/blog/2016/02/25/tsdb-list-part-1/)我们已经介绍了一些常见的TSDB，这里我们再对剩余的一些TSDB做些简单介绍。

## 10. Geras

| - | - | - |
|---|---|---|
| 主页 | http://1248.io/geras.php |  |
| License | 商业 |  |

Geras是一个专注于IoT领域（当然不仅限于传感器采集到的数据）可扩展的、分布式的时序列数据库，用于帮助用户进行快速分析。Geras是一个SaaS服务，但是你也可以购买软件自己部署、托管。Geras提供了免费版，它不对数据存储的时间和数据量做任何限制，只是不提供SLA而已。不过他们的[SaaS主页](http://geras.1248.io/)我却一直没打开过，貌似该DNS记录已经不存在了，官方相关资料也不多，真怀疑这个项目已经废弃了，该产品已经演化为新产品。

Geras速度非常快，它支持对任何时间精度进行Rollup，也支持保存原始数据的精度。它专门为写操作进行了优化，可以接收海量设备的数据输入。

Geras运行在容错、分布式数据存储之上，支持水平扩展，可以在不停止服务的前提下对系统容量进行调整。

Geras也提供了一套可视化界面，方便的对设备、传感器等进行管理，对metric进行图表的可视化等。

Geras支持多种技术来进行数据存储、查询和展示，比如HTTPS、JSON、RESTful、SenML、MQTT、HyperCat，设备数据可以通过HTTP POST或者轻量MQTT发布。

## 11. Akumuli

| - | - | - |
|---|---|---|
| 主页 | https://github.com/Akumuli/akumuli |  |
| 编写语言 | C++ |  |
| License | Apache License, Version 2.0 |  |
| 项目创建时间 | 2014/12 |  |
| 最新版 | pre-release | 2015/10/18 |
| 活跃度 | 活跃 |  |
| 文档 | 简单 |  |

Akumuli名称来自accumulate，是一个数值型时间序列数据库，可以实时存储、处理时序列数据。

它的特点如下：

* 基于日志结构的存储
* 支持无序数据
* 实时压缩
* 基于HTTP的JSON查询支持
* 支持面向行和面向列的存储
* 将最新数据放入内存存储
* 通过metric和tag组织时序列数据，并且可以通过tag进行join操作。
* Resampling (PAA transform), sliding window 
* 通过基于TCP或UDP的协议接收数据（支持百万数据点每秒）
* Continuous queries (streaming)

Akumuli由两部分组成：存储引擎和服务程序。目前只有存储引擎被实现了，而且存储引擎可以在没有服务程序的情况下作为嵌入式数据库单独使用（类似sqlite），也就是说，你得通过自己编写程序使用它的API来实现数据读写。

Akumuli的数据协议基于Redis，它的数据格式也和前面看到的不太一样，比如：

```
+balancers.memusage host=machine1 unit=Gb
+20141210T074343.999999999
:31
```

每条数据由三部分组成，第一部分称为ID，可以是数值型（以“：”开始），或者是字符串（以“+”开始），比如这个例子中，`cpu host=machine1 region= europe`就是ID，它由两部分组成，`cpu`是metric，`host`和`region `则被称为key。

数据的第二部分可以是时间戳（:1418224205），这里的格式为ISO 8601（+20141210T074343.999999）

第三部分为值部分。

类似上面的数据结构，它的query也比较容易理解：

```
{
    "where": {
        "region": [ "europe", "us-east" ]
    },
    "metric": "cpu",
    "range": {
        "from": "20160102T123000.000000",
        "to":   "20160102T123010.000000" }
}

```

Akumuli目前还处于开发状态，不适合在生产环境下使用。

## 12. Atlas

| - | - | - |
|---|---|---|
| 主页 | https://github.com/Netflix/atlas |  |
| 编写语言 | Scala |  |
| License | Apache License Version 2.0 |  |
| 项目创建时间 | 2014/11 |  |
| 最新版 | v1.5.0-rc.4 | 2016/02 |
| 活跃度 | 活跃 | 来自于Netflix |
| 文档 | 详细 |  |

这个和波士顿动力的机器人Atlas，以及其他很多知名软件（HashiCorp、O'reilly）都重名了。

Atlas用于存储带维度信息的时序列数据库，由Netflix开发，用于实时分析。Atlas使用了基于内存的存储，速度非常快。在2011年的时候，Netflix使用自己开发的Epic工具来管理时序列数据，这是一个采用perl、RRDTool和MySQL构建的系统，但是随着数据量的增大（2M->1.2B），所以他们创建了这个新项目。

如果说商业智能（business intelligence）是从数据基于时间分析出趋势的话，那么Atlas可以说是一种运维智能（operational intelligence），能给用户描述出当前系统所发生的所有情况。

Atlas的设计目标主要有3点：

* 通用API
* 可扩展
* 维度支持

Atlas具备和OpenTSDB以及InfluxDB类似的metric/datapoint/tag的概念，同时它也有自己独自的特性，比如增加了步长（step-size）的概念。

Atlas采用了类似RRDtool的查询语法，它们都通过一种对URL有好的语法来支持复杂的数据查询。

一句话点评：背靠大树好乘凉，可以试试。

## 13. Blueflood

| - | - | - |
|---|---|---|
| 主页 | http://blueflood.io/ |  |
| 编写语言 | Java |  |
| License | Apache License, Version 2.0 |  |
| 项目创建时间 | 2013 |  |
| 最新版 | 1.0.2123 | 2016/01/02 |
| 活跃度 | 活跃 | 来自Rackspace |
| 文档 | 比较详细 |  |

Blueflood由Rackspace的Cloud Monitoring团队创建，用于管理Cloud Monitoring系统产生的metric数据。如果你不属性Rackspace可以去网上了解一下，它也是比较大的云计算公司。

除了自己使用，Rackspace还提供了免费的Blueflood-as-a-Service，此外还有一些大公司也在准备使用Blueflood。

Blueflood特点如下：

* built on top of Cassandra
* 基于HTTP API/JSON的数据采集（ingest）和读取
* 支持数值、布尔和字符串metric data
* 多租户
* 水平扩展

Blueflood主要由以下模块构成：

* Ingest - 采集/写入数据
* Rollup - 聚合计算
* Query - 处理用户查询

不过遗憾的是Blueflood现在没有一个类似tag的多维度方案。

一句话点评：背靠大树好乘凉，可以试试。

## 14. Gnocchi

| - | - | - |
|---|---|---|
| 主页 | http://docs.openstack.org/developer/gnocchi/ |  |
| 编写语言 | Python |  |
| License |  Apache License Version 2.0 |  |
| 项目创建时间 | 2014 |  |
| 最新版 | 2.0.0 |  |
| 活跃度 | 活跃 | 来自OpenStack |
| 文档 | 一般丰富 |  |

Gnocchi是OpenStack项目的一部分，但它也能独立工作。

Gnocchi是一个多租户的时间序列、metric和资源（resource）数据库。它提供了一组HTTP REST API来创建和管理数据。

Gnocchi以在云就算环境中存储大量metric信息为设计出发点，为用户提供metric和资源展示功能，同时它也很容易扩展。

Gnocchi特点如下：

* HTTP RES接口
* 水平扩展
* Metri聚合
* 支持批处理
* 支持归档功能
* 按metric值查找（这个在TSDB很少见）
* 多租户支持
* 对Grafan的支持
* 支持Statsd协议

Gnocchi后端存储支持一下4种方式：

* File
* Swift
* Ceph (推荐方式)
* InfluxDB (实验功能)

前三种方式基于一个名为Carbonara的库，这个库负责对是序列数据的管理，因为这三种存储方案本身并不关心数据的特点。InfluxDB前面已经说过，本身就是一个TSDB。


## 15. Newts

| - | - | - |
|---|---|---|
| 主页 | https://github.com/OpenNMS/newts |  |
| 编写语言 | Java |  |
| License | Apache License Version 2.0 |  |
| 项目创建时间 | 2014 | 1.0 release |
| 最新版 | 1.3.3 | 2016/01 |
| 活跃度 | 一般 |  |
| 文档 | 一般 |  |

Newts是基于Cassandra的时序列数据库。正如其名所示，它是一个“New-fangled Timeseries Data Store”。它的开发者是OpenNMS，这也是一个知名的开源网络监控和管理平台。

Newts基于Cassandra，对写优化、完全分布式，吞吐量非常高；通过将类似的metric分组存储、让写和读更高效。

## 16. SiteWhere

| - | - | - |
|---|---|---|
| 主页 | http://www.sitewhere.org/ |  |
| 编写语言 | Java |  |
| License | CPAL 1.0 |  |
| 最新版 | 1.6.1 |  |
| 活跃度 | 活跃 |  |
| 文档 | 详细 |  |


SiteWhere是一个开源的IoT开放平台，帮助用户快速将IoT应用推向市场。SiteWhere提供了一套完整的设备管理解决方案，通过MQTT、AMQP、Stomp和其他协议连接设备，支持自注册、REST和批处理方式注册设备。

SiteWhere也提供了可扩展的大数据解决方案，底层采用经优化的MongoDB和HBase，为存储设备时间提供了时序列数据库，且经过Hortonworks和Cloudera的测试。

SiteWhere还内嵌了Siddhi用于Complex Event Processing （CEP），和Azure EventHub、Apache Solr以及Twilio的集成，提供了Android和Arduino平台开发用的SDK。

SiteWhere的部署方式也很灵活，支持公有云、私有机房，Ubuntu Juju和Docker的部署方式。

SiteWhere也支持多租户，不同的租户数据分开存储，还能为不同的租户提供不同的处理引擎，租户的启动、停止互不影响。

SiteWhere的service provider interfaces（SPIs）提供了平台的核心对象模型，第三方可以对平台进行扩展。

一句话点评：开源、功能强大、一体化方案。作为IoT平台，SiteWhere算是这几个中不错的。

建议你接着看一下它的 [System Overview](http://documentation.sitewhere.org/overview.html) 和 [System Architecture](http://documentation.sitewhere.org/architecture.html) 以对这个系统有更深入的了解。

## 17. TempoIQ

| - | - | - |
|---|---|---|
| 主页 | https://www.tempoiq.com/ |  |
| License | 商业 |  |

TempoIQ也是一个IoT平台服务，它能帮助用户快速、灵活的的创建IoT应用，提供了实时的数据分析、报警、仪表盘、报告功能。2014年7月从TempoDB改名为TempoIQ，它的很多组件都有IQ结尾，代表智商很好？

TempoIQ主要由以下几个部分组成：

* CloudIQ

TempoIQ采用第四代CoDA（Context Delivery Architecture）架构，用户可以不必关系复杂的基础设施就可以部署IoT应用。

* ConnectIQ

基于数据事件的API，用于连接任何设备，支持灵活的事件数据模型：REST API、HTTPs和MQTT。它不关心设备来源，可以及时进行数据流处理，不需要提前安装和配置。

* DataIQ

对所有IoT数据和分析报告进行安全、持久的存储，并提供报告下载功能。

* AnalyzeIQ

用户可以创建分析流（custom analytics streams）来查洞悉IoT数据实时状态，自动将数据存储到DataIQ。并可以创建报警来持续监控分析流，当有超预期的改名或者达到致命条件时进行实时报警。

* ViewIQ

用户使用ViewIQ可以创建实时的IoT仪表盘、应用和可视化组件，而不需要编码工作。

## 18. Riak TS

| - | - | - |
|---|---|---|
| 主页 | http://basho.com/products/riak-ts/ |  |
| 编写语言 | Erlang |  |
| License | 商业 |  |
| 最新版 | 1.1.0 | 2016/01/14 |
| 活跃度 |  | 商业支持 |


Riak作为NoSQL和K/V存储可能更有名，而Riak TS是一个为时序列和IoT数据进行了优化的NoSQL数据库软件。在官方主页上写道：“Riak TS is engineered to be faster than Cassandra”。

由于其非开源性，网上（包括官网）详细资料都不是特别多。

## 19. Cyanite

| - | - | - |
|---|---|---|
| 主页 | http://cyanite.io/ |  |
| 编写语言 | Clojure |  |
| License | MIT |  |
| 项目创建时间 | 2013年 |  |
| 最新版 | | 还没有正式release |
| 活跃度 | 活跃 |  |
| 文档 | 详细 |  |


Cyanite是一个用于接收和存储是序列数据的守护进程，它的设计目标是兼容Graphite生态系统。

Cyanite默认使用Apache Cassandra来存储是序列数据，它的特点如下：

* 可扩展性

基于Cassandra，Cyanite可以实现高可用、具有弹性，以及低延迟的存储。

* 兼容性

由于Graphite已经成为事实上管理时序列数据的标准，不管是使用graphite-web还是Grafana。Cyanite将会尽可能的沿用与这些生态系统的兼容以提供简单地交互模式。

Cyanite是一个典型的流式处理模型，它接收数据，对数据进行标准化，然后进行输出。它的交互图如下所示：

![](/images/2016/02/tsdb-series/cyanite_architecture.png)

它的工作流程如下：

* 每条输入都会产生规范化的metrics，并添加到消息队列
* 核心引擎从队列取出数据并处理。
* 通过存储和索引组件进行时序列数据的存储和metric名的所索引
* API组件用于引擎和其他查询、存储模块的交互)

Cyanite的输入模块支持Carbon（Graphite组件），而Kafka,和Pickle则还在计划中。

索引（Index）模块则用于对metric名进行索引和查询。这一模块有两个实现方式：

* memory用于在内存中存储反向索引
* elasticsearch用于在elasticsearch中存储metric名（path-names）

一句话总结：新生事物、值得关注。

## 20. 总结

这里只是简单的罗列了一些项目，以及他们的简单说明。

在后续的文章中，我们还会选择一些常见的TSDB进行实例讲解。


## 相关阅读

这是本系列文章的其他部分：

* [时序列数据库武斗大会之什么是TSDB](/blog/2016/02/18/tsdb-intro/)
* [时序列数据库武斗大会之TSDB名录 Part 1](/blog/2016/02/25/tsdb-list-part-1/)
* 时序列数据库武斗大会之TSDB名录 Part 2

