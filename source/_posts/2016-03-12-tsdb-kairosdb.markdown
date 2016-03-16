---
layout: post
title: "时序列数据库武斗大会之KairosDB篇"
date: 2016-03-12 23:24:46 +0800
comments: true
categories: 
---


## 简介

按照官方的说明，KairosDB是一个“Fast Time Series Database on Cassandra”，即基于Cassandra的告诉时序列数据库。

## 特点

### 数据采集
数据可以通过多种协议写入KairosDB，比如Telnet的按行写入，HTTP API，Graphite以及批处理导入。此外，还可以使用或者自己编写插件。

### 存储

KairosDB 采用了 Cassandra 作为数据存储方式，Cassandra 也是一个比较流行的NoSQL数据库，很多开源软件基于此数据库。

### Rest API

KairosDB提供了REST API，已完成对metric名称，tag等的查询，当然，也少不了存储和查询数据点（data points）。

### 自定义数据类型（Custom Data）

KairosDB支持存储和聚合自定义数据类型。默认情况下KairosDB支持long、double和字符串的value，这比OpenTSDB要丰富一些。

### 分组和聚合

作为数据分析系统，分组和聚合则是必不可少的功能。
[KairosDB的聚合（也就是down samples）](http://kairosdb.github.io/docs/build/html/restapi/Aggregators.html)功能，支持的标准函数有min、max、sum、count、mean、histogram、gaps等，而且都非常实用。

比如percentile，可以计算一个指标值大概的百分比位置，非常适合存储类似“你打败了xx%的人”这种需求场景。

### 支持工具

KairosDB提供了进行数据导入导出的命令行工具。根据官方文档的说明，在一台分配了2Gig内存的SSD Cassandra上，1秒钟能导入	13万条数据。


### 插件机制

KairosDB也提供多种基于Guice的插件机制来进行扩展（data point监听器，数据存储，协议处理等。）

KairosDB是从OpenTSDB fork过来的，因此最初它是支持HBase的，不过现在HBase已经不能完全支持KairosDB所需的特性，将来会取消对HBase的支持。

# 入门KairosDB

## 安装KairosDB

这里我们以当前最新的1.1.1版本为例进行说明。

首先，需要确保你的JAVA_HOME已经设置好了，且Java版本高于1.6。

```
$ echo $JAVA_HOME
/Library/Java/JavaVirtualMachines/jdk1.8.0_45.jdk/Contents/Home
```

然后需要到[GitHub](https://github.com/kairosdb/kairosdb/releases)上去下载安装包。我用的是OS X系统，因此我选择了[kairosdb-1.1.1-1.tar.gz](https://github.com/kairosdb/kairosdb/releases/download/v1.1.1/kairosdb-1.1.1-1.tar.gz) （注意：点击这个链接即可下载）

解压后可以看看它的配置文件`conf/kairosdb.properties`，有一些东西适合OpenTSDB一样的，比如4242端口。

KairosDB集成了jetty，你可以通过jetty访问WEB UI，而且还支持添加SSL支持，这样安全性上比OpenTSDB高了一个层级。

配置文件中还能对Cassandra进行设置，比如服务器地址、keyspace等。不过默认的话KairosDB使用H2作为数据存储，这样在开发环境下我们就不必配置Cassandra了。这里我们也以H2为例来初步认识一下KairosDB，这也是KairosDB的默认配置。


所以在这个例子里，我们不必修改配置文件，直接启动KairosDB即可：

```
$ bin/kairosdb.sh run
# 或者
$ bin/kairosdb.sh start
```
其中`run`参数会以前台运行的方式启动KairosDB，而`start`则以后台进程的方式启动KairosDB。

停止KairosDB只需要运行`bin/kairosdb.sh stop`就可以了。

## 写入数据

和OpenTSDB一样，KairosDB也支持基于telnet和HTTP API的方式写入数据。

### Telnet

Telnet的方式数据格式很简单：

```
put <metric_name> <time-stamp> <value> <tag> <tag>... \n
```

这里我们就不做演示了。

### HTTP API
只需要发送JSON数据到`http://localhost:8080/api/v1/datapoints`就可以了。

这是我们写入测试数据的方法：

```
$ curl -v -H "Content-type: application/json" -X POST  http://localhost:8080/api/v1/datapoints -d '
[{
    "name": "cpu.load.1",
    "timestamp": 1453109876000,
    "type": "double",
    "value": 0.32,
    "tags":{"host":"test-1"}
},
{
    "name": "cpu.load.1",
    "timestamp": 1453109876000,
    "type": "double",
    "value": 0.21,
    "tags":{"host":"test-2"}
}]
'
* Connected to localhost (::1) port 8080 (#0)
> POST /api/v1/datapoints HTTP/1.1
> Host: localhost:8080
> User-Agent: curl/7.43.0
> Accept: */*
> Content-type: application/json
> Content-Length: 262
> 
* upload completely sent off: 262 out of 262 bytes
< HTTP/1.1 204 No Content
< Access-Control-Allow-Origin: *
< Pragma: no-cache
< Cache-Control: no-cache
< Expires: 0
< Content-Type: application/json; charset=UTF-8
< Server: Jetty(8.1.16.v20140903)
< 
* Connection #0 to host localhost left intact

```
从服务器返回结果我们可以看到，HTTP 204状态码，也是KairosDB成功写入数据的结果。

## 查询数据

同样KairosDB提供了查询用API：

```
$ curl -H "Content-type: application/json" -X POST  http://localhost:8080/api/v1/datapoints/query -d '
{
  "metrics": [
    {
      "tags": {},
      "name": "cpu.load.1",
      "group_by": [
        {
          "name": JSON"tag",
          "tags": [
            "host"
          ]
        }
      ],
      "aggregators": [
        {
          "name": "sum",
          "align_sampling": true,
          "sampling": {
            "value": "1",
            "unit": "minutes"
          }
        }
      ]
    }
  ],
  "cache_time": 0,
  "start_absolute": 1453046400000,
  "end_absolute": 1453132800000,
  "time_zone": "Asia/Chongqing"
}' | jq .
```

注意上面命令最后的jq，这是用来对JSON数据进行格式化的工具。

最终结果可能像下面一样：

```
{
  "queries": [
    {
      "sample_size": 2,
      "results": [
        {
          "name": "cpu.load.1",
          "group_by": [
            {
              "name": "tag",
              "tags": [
                "host"
              ],
              "group": {
                "host": "test-1"
              }
            },
            {
              "name": "type",
              "type": "number"
            }
          ],
          "tags": {
            "host": [
              "test-1"
            ]
          },
          "values": [
            [
              1453109876000,
              0.32
            ]
          ]
        },
        {
          "name": "cpu.load.1",
          "group_by": [
            {
              "name": "tag",
              "tags": [
                "host"
              ],
              "group": {
                "host": "test-2"
              }
            },
            {
              "name": "type",
              "type": "number"
            }
          ],
          "tags": {
            "host": [
              "test-2"
            ]
          },
          "values": [
            [
              1453109876000,
              0.21
            ]
          ]
        }
      ]
    }
  ]
}

```
## WEB UI

KairosDB自带了一个Web界面，你可以通过 http://localhost:8080 访问。不过这个UI主要是以开发为目的的，可以看到查询的JSON文本，方便调试，比较直观。默认的UI使用了Flot来画图，如果你愿意，也可以使用Highcharts替换。


## Library

KairosDB目前有一个单独的[Java Client](https://github.com/kairosdb/kairosdb-client)，在官网还有一些其他语言的客户端，比如Python、PHP等。

由于是Java客户端，所以还是很容易上手的。比如写入数据：

```
MetricBuilder builder = MetricBuilder.getInstance();
builder.addMetric("metric1")
        .addTag("host", "server1")
        .addTag("customer", "Acme")
        .addDataPoint(System.currentTimeMillis(), 10)
        .addDataPoint(System.currentTimeMillis(), 30L);
HttpClient client = new HttpClient("http://localhost:8080");
Response response = client.pushMetrics(builder);
client.shutdown();
```

读取数据：

```
QueryBuilder builder = QueryBuilder.getInstance();
builder.setStart(2, TimeUnit.MONTHS)
       .setEnd(1, TimeUnit.MONTHS)
       .addMetric("metric1")
       .addAggregator(AggregatorFactory.createAverageAggregator(5, TimeUnit.MINUTES));
HttpClient client = new HttpClient("http://localhost:8080");
QueryResponse response = client.query(builder);
client.shutdown();

```
这应该会非常方便，开发起来比OpenTSDB要快不少了。

## 其他API

KairosDB竟然支持metric删除功能，这个功能会有多少人需要呢？

列出metric名、tag列表、列出tag值，说不定有人会喜欢，比如在输入框自动提示灯功能，可能需要这些元数据。

### 列出指标名

这里除了`cpu.load.1`是我们自己写入的metric，其余的都是KairosDB自己的指标数据。

```
$ curl http://localhost:8080/api/v1/metricnames | jq .

  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   501    0   501    0     0  45058      0 --:--:-- --:--:-- --:--:-- 50100
{
  "results": [
    "kairosdb.datastore.query_time",
    "kairosdb.protocol.telnet_request_count",
    "kairosdb.http.ingest_count",
    "kairosdb.datastore.query_row_count",
    "cpu.load.1",
    "kairosdb.protocol.http_request_count",
    "kairosdb.http.ingest_time",
    "kairosdb.jvm.thread_count",
    "kairosdb.jvm.total_memory",
    "kairosdb.jvm.max_memory",
    "kairosdb.metric_counters",
    "kairosdb.jvm.free_memory",
    "kairosdb.datastore.query_sample_size",
    "kairosdb.datastore.query_collisions",
    "kairosdb.http.query_time",
    "kairosdb.http.request_time"
  ]
}

```
### 列出tag key

这个API能列出系统中所有的tag key。不过遗憾的是它不支持只列出某一给定指标的所有tag key。

```
$ curl http://localhost:8080/api/v1/tagnames | jq .

  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100    67    0    67    0     0   4188      0 --:--:-- --:--:-- --:--:--  4466
{
  "results": [
    "method",
    "metric_name",
    "query_index",
    "request",
    "host"
  ]
}

```
### 列出tag value

这个API能列出系统中所有的tag value。同样遗憾的是它也不支持只列出某一给定指标的所有tag value。

所以这两个API几乎可以说是然并卵、无鸟用。

```
$ curl http://localhost:8080/api/v1/tagvalues | jq .

  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   163    0   163    0     0   5011      0 --:--:-- --:--:-- --:--:--  5093
{
  "results": [
    "1",
    "lius-MacBook-Pro.local",
    "tagnames",
    "/datapoints/query",
    "test-1",
    "test-2",
    "metricnames",
    "query",
    "tags",
    "version",
    "datapoints",
    "putm",
    "cpu.load.1"
  ]
}

```

## 总结

KairosDB毕竟是OpenTSDB的一个fork，因此根本上的功能都差不多，而且随着OpenTSDB对Cassandra的支持，感觉KairosDB也该相比OpenTSDB没有什么太大的优势。


## 相关阅读

这是本系列文章的其他部分：

* [时序列数据库武斗大会之什么是TSDB](/blog/2016/02/18/tsdb-intro/)
* [时序列数据库武斗大会之TSDB名录 Part 1](/blog/2016/02/25/tsdb-list-part-1/)
* [时序列数据库武斗大会之TSDB名录 Part 2](/blog/2016/03/01/tsdb-list-part-2/)
* [时序列数据库武斗大会之OpenTSDB篇](/blog/2016/03/05/tsdb-opentsdb/)
* 时序列数据库武斗大会之KairosDB篇
