---
title: elasticsearch入门
author: kiosk
date: 2022-06-18 13:52:18
draft: true
tags:
  - elasticsearch
categories:
  - devops
---

> 前言：最近打算将自己的一些服务尽可能的迁移到 k8s 集群里，ELK 就是其中的一个，后面可以的话将本博客的 搜索功能从 algolia 迁移到 自己的ES 里。搜索这个东西的确很有意思的，在没有自己的 OLAP 分析平台之前，ES 的确是最佳选择。值得好好学习一下。



前面已经有文章介绍 k8s 集群的搭建方法和ES的部署方式了。

- [ubuntu20.04 部署 kubernetes（k8s）](https://kiosk007.top/post/%E6%90%AD%E5%BB%BA%E6%88%91%E7%9A%84elk-7-2/)

# **Elasticsearch 简介**

**Elasticsearch** 是一个分布式、RESTful 风格的搜索和数据分析引擎，能够解决不断涌现出的各种用例。作为 Elastic Stack 的核心，它集中存储您的数据，帮助您发现意料之中以及意料之外的情况。

<br/>

# 快速入门

## 基本概念

- **节点 Node、集群 Cluster 和 分片 Shard**

ElasticSearch 是分布式数据库，允许多台服务器协同工作，每台服务器可以运行多个实例。单个实例称为一个节点（node），一组节点构成一个集群（cluster）。分片是底层的工作单元，文档保存在分片内，分片又被分配到集群内的各个节点里，每个分片仅保存全部数据的一部分。

<br/>

- **索引 Index、类型 Type 和 文档 Document**

索引是一类文档的结合。而文档是具体的一条数据。对比我们比较熟悉的关系型数据库如下：

| RDBMS(MySQL) | Elasticsearch |
| :----------: | :-----------: |
|    Table     |  Index(Type)  |
|     Row      |   Doucment    |
|    Column    |     Field     |
|    Schema    |    Mapping    |
|     SQL      |      DSL      |

<br/>



## 使用 Restful API 与 Elasticsearch 交互

elasticsearch 本身使用 Restful API 通过端口 9200进行交互。其基本组成如下

```bash
curl -X<VERB> '<PROTOCOL>://<HOST>:<PORT>/<PATH>?<QUERY_STRING>' -d '<BODY>'
```

CRUD 具体内容可以参考 [这篇文章](https://juejin.cn/post/6932481200925868040)

<br/>

## 倒排索引

倒排索引是整个 ES 的核心，正常的搜索以一本书为例，应该是由 “目录 -> 章节 -> 页码 -> 内容” 这样的查找顺序，这样是正排索引的思想。

<br/>

>  但是设想一下，我在一本书中快速查找“elasticsearch”这个关键字所在的页面该怎么办？

<br/>

倒排索引的思路是通过单词到文档ID的关系对应。

![es_inverted_index](https://img1.kiosk007.top/static/images/blog/es_inverted_index.png)

倒排索引包含两个部分：

- 单词词典（Term Dictionary）：记录所有文档的单词，记录单词到倒排列表的关联关系（单词词典一般比较大，通过 B+ 树或哈希拉链法实现，以满足高性能的插入与查询）

- 倒排列表（Posting List）：记录了单词对应的文档结合，由倒排索引组成。

  - 文档ID
  - 词频 TF - 该单词在文档中分词的位置。用于语句搜索
  - 位置（Position）- 单词在文档中分词的位置，用于语句搜索
  - 偏移（Offset）- 记录单词的开始结束位置，实现高亮显示。

  <br/>

## Analyzer 分词

上面的倒排索引我们看到了，其实一个文档中的内容是会被倒排索引的，那也就是说一个长文本会被分词，Elasticsearch 中的 Analyzer 就是这样的一个角色。

<br/>

Analyzer 由三部分组成，**Character Filters**（针对原始文本处理，例如去除html）/ **Tokenizer**（按照规则切分为单词）/ **Token Filter**（将切分的单词进行加工，小写，删除 stop words）



比如如下的 standard Analyzer

![es_analyzer](https://img1.kiosk007.top/static/images/blog/es_analyzer.png)



这里比较特殊的是中文分词，中文词语在不同句子中的上下文不太一样。

这里以 icu analyzer 为例，首先使用前需要安装插件。

```
./bin/elasticsearch-plugin install analysis-icu
```

<br/>

| 内容                                                         |                             分词                             |
| ------------------------------------------------------------ | :----------------------------------------------------------: |
| ![analyzer_chinese](https://img1.kiosk007.top/static/images/blog/analyzer_chinese.webp) | ![analyzer_icu](https://img1.kiosk007.top/static/images/blog/analyzer_icu.png) |

haha :rofl: ，icu 在这个困扰 NLP 多年的案例上也栽了跟头，除此之外还有 [IK](https://github.com/medcl/elasticsearch-analysis-ik)、[THULAC](https://github.com/microbun/elasticsearch-thulac-plugin)可以试一试

<br/>

 ## Search API

搜索是 Elasticsearch 最重要的一个功能，和搜索引擎一样，搜索可以加入一些相关性 （Relevance）逻辑，比如结合业务竞价排序搜索结果。



搜索一般分为 URL Search 和 Request Body Search 。一般用得比较多的是Request Body Search。





具体搜索API这里不做太多介绍，可以网上找一些博客，对基本使用肯定是够用了。

https://juejin.cn/post/6934298179118743566

<br/>

## Dynamic Mapping

Mapping 类似数据库中的 schema 定义，比如定义索引中字段的名称，定义字段的数据类型，如字符串、数组、布尔... 字段、倒排索引的相关配置。



Mapping 会把 JSON 文档映射成 Lucene 所需要的扁平格式。

<br/>

而 Dynamic Mapping 的意义是在写入文档时候，如果索引不存在，会自动创建索引。有了这个机制，我们无需再手动定义 Mappings，Elasticsearch 会自动根据文档信息推算出字段的类型。





相关的文章：

https://www.elastic.co/guide/cn/elasticsearch/guide/current/dynamic-mapping.html

<br/>

当然除了自动创建Mapping，也可以自行显示定义一个 Mapping，方法如下：

```
PUT users
{
	"mappins": {
		"properties": {
			"firstName": {
				"type": "text"
			},
			"lastName": {
				"type": "text"
			}
			"mobile": {
				"type": "text",
				"index": false    // 设置为 false ，不会被倒排索引，更不会被搜索
			}
		}
	}
}
```



索引相关文章：

https://www.cnblogs.com/wupeixuan/p/12514843.html

<br/>



Mapping 中配置自定义 Analyzer

当 Elasticsearch 自带的分词器无法满足时，可以自定义分词器，通过自组合不同的组件实现。甚至可以自定义一些替换规则，比如将文档中的 “&” 替换为 “and”，另外去除停用词，大小写自动转换等。



具体参考：

https://www.elastic.co/guide/cn/elasticsearch/guide/current/custom-analyzers.html

<br/>

## Index 创建管理

索引的相关属性细节非常多，官网有整整一个[章节](https://www.elastic.co/guide/cn/elasticsearch/guide/current/index-management.html)去介绍。例如上面的 Mapping 属性，索引使用的 Analyzer 分词器等等。那么在创建索引时肯定就有模板。

index Templates 帮助你设定 Mapping 和 Settings，并按照一定的规则，自动匹配到新创建的索引之上

- 模板仅在一个索引被新创建时，才会产生作用。修改模板不会影响已创建的索引
- 可以设定多个索引模板，这些设置会被 merge 在一起
- 可以指定 order 的数值，控制 merging 的过程
  



具体参考：

https://blog.csdn.net/xixihahalelehehe/article/details/109595303

https://www.elastic.co/guide/en/elasticsearch/reference/current/index-templates.html