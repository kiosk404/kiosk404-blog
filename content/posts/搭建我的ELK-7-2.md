---
title: 搭建我的ELK 7.12
author: kiosk
date: 2021-03-27 13:52:18
tags:
  - elasticsearch
categories:
  - devops
---
Elasticsearch 是一个实时的分布式搜索分析引擎，它能让你以前所未有的速度和规模，去探索你的数据。

“ELK”是三个开源项目的首字母缩写，这三个项目分别是：Elasticsearch、Logstash 和 Kibana。Elasticsearch 是一个搜索和分析引擎。Logstash 是服务器端数据处理管道，能够同时从多个来源采集数据，转换数据，然后将数据发送到诸如 Elasticsearch 等“存储库”中。Kibana 则可以让用户在 Elasticsearch 中使用图形和图表对数据进行可视化。

<!--more-->

引用官网的一句话：
> Elasticsearch 是一个分布式、RESTful 风格的搜索和数据分析引擎，能够解决不断涌现出的各种用例。 作为 Elastic Stack 的核心，它集中存储您的数据，帮助您发现意料之中以及意料之外的情况。



# 简介

<img src="https://img1.kiosk007.top/static/images/elk/elk.png" style="height:500px">

**ElasticSearch 的目录结构**

| 目录  |	配置文件 |	描述 |
| --------   | -----:  | :----:  |
| bin  |	| 	脚本文件，包括启动elasticsearch，安装插件，运行统计数据等 
| config	| elasticsearch.yml	| 集群配置文件，user，role based 相关配置
| JDK	| |	java 运行环境 |
| data	| path.data	| 数据文件 |
| lib	|	| java 类库 |
| logs	| path.log	| 日志文件 |
| modules | |		包含所有ES模块 |
| plugins | |		包含所有已经安装的插件 |


# 开始安装

官方文档 Set up Elasticsearch 有各个 OS 的安装指导，页面 [Installing Elasticsearch](https://www.elastic.co/guide/en/elasticsearch/reference/7.3/install-elasticsearch.html) 中提供了多种安装包对应的指导链接！

本文选择绿色安装包的的方式（tar.gz）安装。

- 安装环境： ubuntu 20.04
- 下载链接： [华为镜像站]( https://mirrors.huaweicloud.com/elasticsearch/) 速度能快一点


说明：ElasticSearch使用java语言开发，所以默认需要安装并配置JDK，设置 JAVA_HOME, 但是从 7.0 开始，ElasticSearch 内置了Java环境，无需再安装。另外ES启动不能使用root用户

内核参数修改（32C + 128G参考）

``` bash
#修改文件描述符数量
grep "* - nofile 512000" /etc/security/limits.conf || echo  "* - nofile 512000"  >> /etc/security/limits.conf  

#修改最大打开进程数数量
grep "work - nproc unlimited" /etc/security/limits.conf || echo "elasticsearch - nproc unlimited"   >> /etc/security/limits.conf  
#配合es mem lock，centos6无须添加
grep "* soft memlock unlimited" /etc/security/limits.conf || echo "* soft memlock unlimited"   >> /etc/security/limits.conf  

#配合es mem lock，centos6无须添加
grep "* hard memlock unlimited" /etc/security/limits.conf || echo "* hard memlock unlimited"   >> /etc/security/limits.conf  

#修改系统文件描述符
grep "fs.file-max = 1024000" /etc/sysctl.conf || echo "fs.file-max = 1024000"  >> /etc/sysctl.conf 

#修改程序最大管理的vm
grep "vm.max_map_count = 262144" /etc/sysctl.conf || echo "vm.max_map_count = 262144"  >>  /etc/sysctl.conf  

grep  "vm.min_free_kbytes = 2097152" /etc/sysctl.conf || echo "vm.min_free_kbytes = 2097152"  >>  /etc/sysctl.conf

grep  "vm.zone_reclaim_mode = 0" /etc/sysctl.conf || echo "vm.zone_reclaim_mode = 0"  >>  /etc/sysctl.conf

sysctl -p

swapoff -a   #关闭虚拟内存
```

## 1. 安装 elasticsearch

``` bash
wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-7.12.0-linux-x86_64.tar.gz
tar -xf elasticsearch-7.12.0-linux-x86_64.tar.gz -C ~
cd ~/elasticsearch-7.12.0
./bin/elasticsearch  # 启动
```
启动成功后访问本地的 9200 端口，可以看到

``` bash
$ curl 127.0.0.1:9200
{
  "name" : "k8s-master",
  "cluster_name" : "elasticsearch",
  "cluster_uuid" : "sEn3TgEVSnW4kHpIAU1-5Q",
  "version" : {
    "number" : "7.12.0",
    "build_flavor" : "default",
    "build_type" : "tar",
    "build_hash" : "78722783c38caa25a70982b5b042074cde5d3b3a",
    "build_date" : "2021-03-18T06:17:15.410153305Z",
    "build_snapshot" : false,
    "lucene_version" : "8.8.0",
    "minimum_wire_compatibility_version" : "6.8.0",
    "minimum_index_compatibility_version" : "6.0.0-beta1"
  },
  "tagline" : "You Know, for Search"
}

```

<p class='div-border red'>如果有安装的错误，参考：</p>

- **seccomp unavailable 错误**
```
解决方法：elasticsearch.yml 配置
bootstrap.memory_lock: false
bootstrap.system_call_filter: false
```

- **max file descriptors [4096] for elasticsearch process likely too low, increase to at least [65536]**
```
解决方法：修改 /etc/security/limits.conf，配置：
hard nofile 80000
soft nofile 80000
```

- **max virtual memory areas vm.max_map_count [65530] is too low**
```
解决方法：修改 /etc/sysctl.conf，添加 ：
vm.max_map_count = 262144
然后 sysctl -p 生效
```

> 安装插件方式：./bin/elasticsearch-plugin install analysis-icu


**ES 相关配置**
- 官网关于配置的内容主要有两处：
  - [Configuring Elasticsearch](https://www.elastic.co/guide/en/elasticsearch/reference/current/settings.html)
  - [Important Elasticsearch configuration](https://www.elastic.co/guide/en/elasticsearch/reference/current/important-settings.html)
- Elasticsearch 主要有三个配置文件：
  - `elasticsearch.yml`：ES的配置文件 [more](https://www.elastic.co/guide/en/elasticsearch/reference/current/important-settings.html)
  - `jvm.options`: ES JVM 参数 [more](https://www.elastic.co/guide/en/elasticsearch/reference/current/jvm-options.html#jvm-options)
  - `log4j2.properties`: ES log 配置 [more](https://www.elastic.co/guide/en/elasticsearch/reference/current/logging.html#logging)
  

默认情况，ES 告诉 JVM 使用一个最小和最大都为 4GB 的堆。但是到了生产环境，这个配置就比较重要了，确保 ES 有足够堆空间可用。
> 但是我的XPS 16G内存。不改堆内存大小的只能起一个实例，再起其他实例，旧的实例总显示 Killed。
修复方式，更改 `./config/jvm.options`
``` bash
-Xms1g 
-Xmx1g
```


**运行多个Elasticsearch 实例**

每个实例的配置文件需要不同，这里降低复杂度，不修改配置文件，而是直接用命令行的形式启动一个集群。

- 启动实例

``` bash
# 启动实例1
./bin/elasticsearch -E cluster.name=myes -Enode.name=node0 \
-E node.master=true -E node.data=false -E node.ingest=false \
-E network.host=127.0.0.1 \
-E http.port=9200 -E transport.tcp.port=9300 \
-E discovery.seed_hosts="127.0.0.1:9300","127.0.0.1:9301","127.0.0.1:9302" \
-E cluster.initial_master_nodes="node0" -d

# 启动实例2 
./bin/elasticsearch -E cluster.name=myes -E node.name=node1 \
-E node.master=true -E node.data=true -E node.ingest=false \
-E path.data=./data/data_node1 -E network.host=127.0.0.1 \
-E http.port=9201 -E transport.tcp.port=9301 \
-E discovery.seed_hosts="127.0.0.1:9300","127.0.0.1:9301","127.0.0.1:9302" \
-d


# 启动实例3 
./bin/elasticsearch -E cluster.name=myes -E node.name=node2 \
-E node.master=true -E node.data=true -E node.ingest=false \
-E path.data=./data/data_node2 -E network.host=127.0.0.1 \
-E http.port=9202 -E transport.tcp.port=9302 \
-E discovery.seed_hosts="127.0.0.1:9300","127.0.0.1:9301","127.0.0.1:9302" \
-d
```

> - 9300端口： ES节点之间通讯使用
> - 9200端口： ES节点 和 外部 通讯使用
> - `discovery.seed_hosts`: 发现设置。有两种重要的发现和集群形成配置，以便集群中的节点能够彼此发现并且选择一个主节点.其中 `discovery.seed_hosts` 是组件集群时比较重要的配置，用于启动当前节点时，发现其他节点的初始列表。
> 当一个已经加入过集群的节点重启时，如果他无法与之前集群中的节点通信，很可能就会报这个错误 master not discovered or elected yet, an election requires at least 2 nodes with ids from。必须至少配置 [discovery.seed_hosts，discovery.seed_providers，cluster.initial_master_nodes] 中的一个。
> - `cluster.initial_master_nodes`: 初始的候选 master 节点列表。初始主节点应通过其 node.name 标识，默认为其主机名。确保 cluster.initial_master_nodes 中的值与 node.name 完全匹配。
<p class='div-border red'>`cluster.initial_master_nodes` 该配置项并不是需要每个节点设置保持一致，设置需谨慎，如果其中的主节点关闭了，可能会导致其他主节点也会关闭。因为一旦节点初始启动时设置了这个参数，它下次启动时还是会尝试和当初指定的主节点链接，当链接失败时，自己也会关闭！
因此，为了保证可用性，预备做主节点的节点不用每个上面都配置该配置项！保证有的主节点上就不设置该配置项，这样当有主节点故障时，还有可用的主节点不会一定要去寻找初始节点中的主节点！</p>
- 详细资料参考：

  [Bootstrapping a cluster](https://www.elastic.co/guide/en/elasticsearch/reference/master/modules-discovery-bootstrap-cluster.html)

  [Discovery and cluster formation settings](https://www.elastic.co/guide/en/elasticsearch/reference/current/modules-discovery-settings.html)

> 在新版 7.x 的 ES 中，对 ES 的集群发现系统做了调整，不再有 discovery.zen.minimum_master_nodes 这个控制集群脑裂的配置，转而由集群自主控制，并且新版在启动一个新的集群的时候需要有cluster.initial_master_nodes 初始化集群主节点列表。如果一个集群一旦形成，你不该再设置该配置项，应该移除它。该配置项仅仅是集群第一次创建时设置的！集群形成之后，这个配置也会被忽略的！
>
> - `discovery.seed_hosts`: 提供群集中符合master节点资格的地址列表


node0 节点仅仅是一个 master 节点，它不是一个数据节点。

先启动 node0 节点，因为它设置了初始主节点的列表。这时候就可以使用 `http://<host IP>:9200/` 看到结果了。然后逐一启动 node1 和 node2。通过访问 `http://127.0.0.1:9200/_cat/nodes` 查看集群是否 OK。
``` bash
$ curl 127.0.0.1:9200/_cat/nodes
127.0.0.1 42 49 58 4.45 2.09 1.37 lmr        * node0
127.0.0.1 42 49 54 4.45 2.09 1.37 cdfhlmrstw - node1
127.0.0.1 42 49 45 4.45 2.09 1.37 cdfhlmrstw - node2
```

## 2. 安装 Kibana

``` bash
wget https://artifacts.elastic.co/downloads/kibana/kibana-7.12.0-linux-x86_64.tar.gz
tar -xf kibana-7.12.0-linux-x86_64.tar.gz
cd kibana-7.12.0-linux-x86_64

```

**启动kibana**

``` bash
# 将kibana改成中文
vim config/kibana.yml
     i18n.locale: "zh-CN"   ## 最后一行
./bin/kibana

```
访问本地的5601端口
查看样例。点击右下角的 `try out sample data` ，可以导入kibana的测试数据。分别是电商网站报表、航空数据、日志
<img src="https://img1.kiosk007.top/static/images/elk/kinana_home.png">
这里分 [Enterprise Search(企业搜索)](https://www.elastic.co/cn/enterprise-search?elektra=home&storm=river1)、[Observability(监控)](https://www.elastic.co/guide/en/observability/7.9/observability-introduction.html)、[Security(安全)](https://www.elastic.co/guide/en/security/7.9/es-overview.html)

- Enterprise Search(企业搜索)：可建立强大的搜索体验，当然是付费滴。
- Observability(监控)：日志、APM、站点SLA监控、指标打点。（支持Nginx、MySQL、Redis等日志）
- Security(安全): 安全相关的解决方案


另外还开以打开 `http://127.0.0.1:5601/app/dev_tools#/console` 控制台，这个是直接对接 ES 的。可在这里直接使用查询语句。


## 3. 安装 Logstash

``` bash
wget https://artifacts.elastic.co/downloads/logstash/logstash-7.12.0-linux-x86_64.tar.gz

```

## 4. 安装 cerebro 

cerebro是专业化项目管理系统，提供一个协作工作环境和项目管理软件，用于处理复杂的视觉材料。它
专为 CGI 和动画工作室、广告公司、电视公司和建筑设计公司而开发。也可以说它是一款Elasticsearch监控工具。

安装
``` bash
wget https://github.com/lmenezes/cerebro/releases/download/v0.9.3/cerebro-0.9.3.tgz
tar -xf cerebro-0.9.3.tgz 
cd cerebro-0.9.3 
./bin/cerebro   # 启动

```
> cerebro 需要 java 才能运行，没有java环境的化，可以执行 `sudo apt install openjdk-11-jdk` 。 Java 11 是 Java 的一个长期支持版本（LTS）。它同时也是 Ubuntu 20.04的默认 Java 开发和运行环境。

访问 `http://127.0.0.1:9000` 浏览器打开。

<img src="https://img1.kiosk007.top/static/images/elk/cerebro.png
 "/>

点击左上方 node，可查看节点情况。

## 4. 测试

下载测试样本 movielens 
``` bash
 wget http://files.grouplens.org/datasets/movielens/ml-20m.zip
 unzip ml-20m.zip
```

开始配置文件
``` bash
input {
  file {
    path => "/home/work/logs/ml-20m/movies.csv"
    start_position => "beginning"
    sincedb_path => "/dev/null"
  }
}
filter {
  csv {
    separator => ","
    columns => ["id","content","genre"]
  }

  mutate {
    split => { "genre" => "|" }
    remove_field => ["path", "host","@timestamp","message"]
  }

  mutate {

    split => ["content", "("]
    add_field => { "title" => "%{[content][0]}"}
    add_field => { "year" => "%{[content][1]}"}
  }

  mutate {
    convert => {
      "year" => "integer"
    }
    strip => ["title"]
    remove_field => ["path", "host","@timestamp","message","content"]
  }
}
output {
   elasticsearch {
     hosts => "http://localhost:9200"
     index => "movies"
     document_id => "%{id}"
   }
  stdout {}
}

```

导入测试数据到ES中
` logstash -f log.conf`
