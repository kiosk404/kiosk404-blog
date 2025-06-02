# elasticsearch 原理及入门


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



## 使用 Restful API 交互

elasticsearch 本身使用 Restful API 通过端口 9200进行交互。其基本组成如下

```bash
curl -X<VERB> '<PROTOCOL>://<HOST>:<PORT>/<PATH>?<QUERY_STRING>' -d '<BODY>'
```

CRUD 具体内容可以参考 [这篇文章](https://juejin.cn/post/6932481200925868040)

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

<br/>

## Elasticsearch Master

Master Node 在 Elasticsearch 集群中的角色是处理创建、删除索引等请求并决定分片被分配到哪个节点等工作。

<br/>

Master 节点非常重要，在部署上需要考虑解决单节点问题。

为一个集群设置多个 Master 节点/ 每个节点只承担 Master 单一角色。

**Master Eligible Nodes 的选主过程**

正常情况下，所有的备选节点会互相 Ping 对方，Node Id 低的节点会成为被选举节点，暂且认为它是Master节点。

如果对某个节点的投票数达到一定值（n/2+1）并且该节点自己也选举自己，那么这个节点就正式成为 master 节点。



其他文章参考：https://juejin.cn/post/6844903975234306062

<br/>

## Primary Shard / Replica Shard

Index 索引是文档的基本机制单位，以 Index 为单位，将文档存放在 shard 这个存储单元上。

Shard 分为 `primary shard` 和 `replica shard`，后者是前者的数据备份。

<br/>

**`primary shards`**：和其它 primariy shard 一起存储 index 中全部文档，每个文档只会被 primariy shard 存储一次，分散在多个Data Node 之上。primariy shard 的目的是用水平扩展方式提高存储容量。 

**`replica shards`**：每个 primariy shard 对应的备份。replica shard 的目的是数据备份，防丢失。

<br/>

`primary shards 的数量在 index 创建时指定，replica shards 数量可以随时更改。`如果修改 primary shard 数量，需要 ReIndex。



{{< admonition type=tip title="分片的设定" open=true >}}

- 如何规划一个索引的主分片数和副本分片数

  - 主分片数设置过小：例如创建了 1 个 Primary Shard 的 Index，如果该索引增长很快，集群无法通过增加节点实现索引数据的扩展
- 主分片数设置过大：导致单个 Shard 容量很小，引发一个节点上有过多分片，影响性能
  - 副本分片设置过多：会降低集群整体的写入性能。

{{< /admonition >}}



其他文章参考：https://juejin.cn/post/6918745075761545223

<br/>





# Elasticsearch 整体架构

![elasticsearch_view](https://img1.kiosk007.top/static/images/blog/elasticsearch_view.png)

- 一个 ES Index 在集群模式下，有多个Node（节点）组成，每个节点就是ES的 instance（实例）
- 每个节点上会有多个 shard（分片），P1 P2 是主分片，R1 R2 是副本分片。
- 每个分片上对应着就是一个 Lucene Index （底层索引文件）
- Lucene Index 是一个统称：
  - 由多个 Segment（段文件，就是倒排索引）组成，每个段文件存储着的就是 Doc 文档。
  - commit point 记录了所有的 segments 的信息

<br/>

**Lucene 索引结构**

![es-th-2-2](https://img1.kiosk007.top/static/images/blog/es-th-2-2.png)

```bash
# 一个 Segment 的内部
$ tree xgChCUdvTZ2G0dyJjGcp6A
xgChCUdvTZ2G0dyJjGcp6A
├── 0
│   ├── _state
│   │   ├── retention-leases-42.st
│   │   └── state-3.st
│   ├── index
│   │   ├── _1e.fdm
│   │   ├── _1e.fdt
│   │   ├── _1e.fdx
│   │   ├── _1e.fnm
│   │   ├── _1e.kdd
│   │   ├── _1e.kdi
│   │   ├── _1e.kdm
│   │   ├── _1e.si
│   │   ├── _1e_1.fnm
│   │   ├── _1e_1_Lucene90_0.dvd
│   │   ├── _1e_1_Lucene90_0.dvm
│   │   ├── _1e_Lucene90_0.doc
│   │   ├── _1e_Lucene90_0.dvd
│   │   ├── _1e_Lucene90_0.dvm
│   │   ├── _1e_Lucene90_0.tim
│   │   ├── _1e_Lucene90_0.tip
│   │   ├── _1e_Lucene90_0.tmd
│   │   ├── _1f.cfe
│   │   ├── _1f.cfs
│   │   ├── _1f.si
│   │   ├── segments_e
│   │   └── write.lock
│   └── translog
│       ├── translog-16.tlog
│       └── translog.ckp
└── _state
    └── state-8.st

```



<br/>

# Lucene 倒排索引核心原理

Lucene 是一个成熟的权威检索库，具有高性能、可伸缩的特点，并且开源、免费。在其基础上开发的分布式搜索引擎便是 Elasticsearch。

![es_struct](https://img1.kiosk007.top/static/images/blog/es_struct.png)



Elasticsearch 的搜索原理简单过程是，索引系统通过扫描文章中的每一个词，对其创建索引，指明在文章中出现的次数和位置，当用户查询时，索引系统就会根据事先的索引进行查找，并将查找的结果反馈给用户的检索方式。



## 倒排索引

倒排索引是整个 ES 的核心，正常的搜索以一本书为例，应该是由 “目录 -> 章节 -> 页码 -> 内容” 这样的查找顺序，这样是正排索引的思想。

<br/>

>  但是设想一下，我在一本书中快速查找 “elasticsearch” 这个关键字所在的页面该怎么办？

<br/>

倒排索引的思路是通过单词到文档ID的关系对应。

![es_inverted_index](https://img1.kiosk007.top/static/images/blog/es_inverted_index.png)

倒排索引包含两个部分：

- 单词词典（Term Dictionary）：记录所有文档的单词，记录单词到倒排列表的关联关系（单词词典一般比较大，通过 B+ 树或哈希拉链法实现，以满足高性能的插入与查询）
- 倒排表（Posting List）：记录了单词对应的文档结合，由倒排索引组成。

  - 文档ID
  - 词频 TF - 该单词在文档中分词的位置。用于语句搜索
  - 位置（Position）- 单词在文档中分词的位置，用于语句搜索
  - 偏移（Offset）- 记录单词的开始结束位置，实现高亮显示。


<br>

## 倒排表（Posting List）

倒排表记录了对应单词（Term Dictionary）所出现的的文档ID等信息。并且为了搜索的时延肯定需要放在内存中，面对海量的文档必然会存在更多量级的倒排表，为了节约空间，肯定是需要一定的压缩算法。



### **FOR：（Frame Of Reference）**

假设某个包含某个 单词 （Term Dictionary）的文档出现了100W次，那么其对应的倒排表就会非常的大，按1个int占用空间为 4 Byte 计算，仅这么倒排表中的一项就要消耗 3.8MB 空间。

![es_posting_list_for1](https://img1.kiosk007.top/static/images/blog/es_posting_list_for1.png)

如上图所示，我们知道一个1个int是4字节，一个字节最大可以存正21亿，1个bit可以存2个数，2个bit可以存4个数（0,1,2,3）。那么假设我们存的都是非常小的数字能否将存储所占空间压下来呢。如果我们只取 posting list 中的数字差值，这将是一个非常小的数字，比如上图是100W个1。这样我们通过只取差值，得到了一个100W个1的列表，并将每个元素只耗费1bit存储了下来。这样可以压缩32倍存储空间。



但事实是，一般没有这么理想的状态。

![es_posting_list_for2](https://img1.kiosk007.top/static/images/blog/es_posting_list_for2.png)

假设实际场景中的 倒排表数组是 `[73, 300, 302, 332, 342, 372]`，原本需要4 * 6 byte = 24byte = 192bit] 。压缩后为 `[73, 227, 2, 30, 11, 29 ]` 。

这些数中227是最大的，需要8bit（227 < 2^8）来盛装，那么每个数值都不会超过8bit，所以需要的大小是6 * 8bit=48bit。因为相比较于原数组，我们引入了一个箱子的概念，那么除了箱子数，我们还需要记录每个箱子的大小，所以需要有一个数来记录箱子大小（假设 8 bit 用来声明 8 bit 区）。那么总共的大小是 48 bit + 8 bit = 56 bit。

<br/>

可以看到压缩后大小由192bit降到了56bit，已经有很大改善了，但是这还不是FOR算法的终点，观察这组数中最大值227,后一位最小值是2，两者相差很大，2实际上只需要1bit来盛装，那么能不能进一步压缩呢？答案是可以，只是不再需要做差值，直接将数组分组，分为：

```
8 bit 区：[73, 227]
5 bit 区：[2, 30, 11, 29]
```

那么占用空间就变成了73,227箱子大小8bit，2,30,11,29中最大30，箱子大小为5bit
因此数组总大小为2*8bit + 4*5bit = 36bit，另外不要忘记这里因为分成两组，还需要单独记录两组箱子的大小值，所以总大小是36bit+2*8bit= 52bit  



<br/>

<!------>

### **RBM：（RoaringBitmap）**

有了FOR算法为什么需要别的算法呢？说明FOR算法本身是有缺陷的，那么思考一下FOR算法的缺陷在哪里？

For 算法处理的数组比较稠密，假设数组分的比较散，差值减后的数字和减之前没有差太多，For 算法就失灵了。

比如下面这个数组

```
[1000, 62101, 131385, 191173, 196658]
```

上面的数组做差值显然没有变化多少，而 RBM的核心就是通过除法来缩减数值的大小，但是并不是直接相除。

比如 196658 的二进制表示为 `0000 0000 0000 0011 0000 0000 0011 0010` 

然后将其高16位和低16位分别转换为10进制：

```
0000 0000 0000 0011 -> 3
0000 0000 0011 0010 -> 50
```

那么那么196658就转换成了`(3, 50)` 的表示形式，其效果就相当于除以2^16,商3余50

![es_posting_list_rbm](https://img1.kiosk007.top/static/images/blog/es_posting_list_rbm.png)

我们发现这个列表的每个元素被拆成了2部分，其中高8位经常出现重复，所以高8位就可以重复利用了，这里称之为 short key。后面对应的低 8 位被称之为 Container，这里是 Array Container，实际上还有 BitMapContainer 和 RunContainer。



{{< admonition type=tip title="这就结束了吗？" open=true >}}

我们观察到，每个 Container 实际上都是一个有序数值数组，是不是能够联想到什么？

1. 数组还能进行压缩吗？
2. 数组能用FOR再压缩吗？
3. 还有其他的压缩方式吗？

{{< /admonition >}}



首先，数组肯定是还可以接着压缩的，**但是为了压缩效率和解码效率，这里不再适合 FOR算法了**。

这里再讲述一下 Container 中的 bitmap Container。 

假设我们要存储10亿条 11 位的电话号码。如果用 Long 类型存储的话， 8Byte * 10亿 大约是 7.45G 。

所以用 Long 存储显然是不行的。但细想一下，11 位的电话号码肯定满足以下特点：

```
10000000000 - 20000000000
```

那么我们用一个 bit 来表示这个数是否存在即可。这需要 10000000000（100亿）个 bit，也就是大概 1MB，假设 13899101122 这个电话号码需要被存储，那么只需要将第 13899101122 个bit 字节置为1即可。

<br/>

当然，依据此原理 Container 本身就可以选择 bitmap 来存储了，但是 bitmap 有个缺点就是无论 Container中有多少数，都要占用 8K 大小，当每个Container 的数量都超过 4096 个时，使用 bitmap 才更划算。



![es_posting_list_contianer](https://img1.kiosk007.top/static/images/blog/es_posting_list_contianer.png)





<br/>

<!---->

## 字典树（Term Dictionary Tree）

倒排索引的核心在于如何快速的依据`查询词`快速的查找到所有的相关文档，我们可以采用 HashMap、TRIE、Binary Search Tree等数据结构来实现。

而 Lucene 采用了一种称为 FST（Finite State Transducer） 的结构来构建词典，这个结构保证了时间和空间复杂度，是Luene的核心功能之一。

### 关于FST（Finite State Transducer）

FST 是一种类似 TRIE 树的算法。

> Trie树，又叫字典树、前缀树（Prefix Tree）,之前写 Web 路由时刚好接触到过前缀树。
>
> - [前缀树路由Router](https://geektutu.com/post/gee-day3.html)
> - [浅谈Trie树](https://zhuanlan.zhihu.com/p/68447845)

那为什么不直接使用 Trie 树这种现成的搜索树算法呢？

假设我们有这样一个 Set： mon、thues、thurs。FSA 是这样的（终点节点4）：

<img src="https://img1.kiosk007.top/static/images/blog/fst-days3.png" alt="fst-days3" style="zoom:150%;" />

相对应的 Trie 则是这样的：

<img src="https://img1.kiosk007.top/static/images/blog/fst-days3-trie.png" alt="fst-days3-trie" style="zoom:150%;" />



使用有限状态转换器在内存消耗上远比 SortedMap 要少，但是在查询过程中需要更多的CPU资源。另外，ES中有一种查询叫做**模糊查询**（fuzzy query），根据搜索词和字段之间的**编辑距离**来判断是否匹配。





参考：

- [关于Lucene的词典FST深入剖析](https://www.shenyanchao.cn/blog/2018/12/04/lucene-fst/)



<br/>

# Elasticsearch索引文档流程

## 文档索引步骤顺序

新建单个文档所需要的步骤顺序：

<img src="https://img1.kiosk007.top/static/images/blog/es-th-2-4.png" style="zoom:150%;" >



1. 客户端向 Node1 发送新建、索引或者删除请求。
2. 节点使用文档的_id 确定文档属于分片0。且P0在Node3 ，所以请求会被转发至Node3
3. Node3 上执行请求，成功后，将请求并行转发至 Node1 和 Node2 的副本分片上，一旦所有的副本分片都报告成功，Node3 向协调节点报告成功，协调节点向客户端报告成功。

<br/>

<!--------->

## 文档索引过程

<img src="https://img1.kiosk007.top/static/images/blog/es-th-2-5.jpeg" alt="es-th-2-5" style="zoom:150%;" />



- 协调节点默认使用文档ID进行计算（也支持通过 routing），为路由提供合适的分片。

```
shard = hash(document_id) % (num_of_primary_shards)
```

- 当分片所在的节点接收到来自协调节点（Coordinating Node）的请求后，将请求写入到 Memory Buffer，然后定时（默认是每隔1s）写入到 Filesystem Cache，**这个从 Memery Buffer 到 Filesystem Cache 的过程叫做 refresh**。
- 假设当 Memory Buffer 和 Filesystem Cache 的数据丢失时，ES 通过 translog 的机制来保证数据的不丢失。其实现机制是接收到请求后，同时也会写入到 translog 中，**当 Filesystem Cache 中的数据写入到磁盘中时，才会清除掉，这个过程叫做 flush**。
- 在 flush 过程中，内存中的缓冲将被清除，内容被写入一个新的 Segment，Segment 的 fsync 将创建一个新的提交点（ Commit Point），并将内容刷新到磁盘，旧的translog将被删除并开始一个新的translog，flush 触发的时机是定时触发（默认30分钟）或者 translog 变得太大（默认512M）时。

<br/>

## 分步查看数据持久化过程

> **write** -> **refresh** -> **flush** -> **merge** 

- **write 过程**

<img src="https://img1.kiosk007.top/static/images/blog/es-th-2-6.png" alt="es-th-2-6" style="zoom:67%;clear:both; display: block; margin: auto" />

一个新文档过来，会存储在 in-memory buffer 内存缓冲区中，顺便会记录 Translog（Elasticsearch 通过事务日志，在每一次对 Elasticsearch 进行操作时记录操作）。

<br/>

**这个时候数据还没有到 segment，搜不到新文档。数据只有被 refresh 后，才可以被搜索到。**

<br/>

- **refresh 过程**

<img src="https://img1.kiosk007.top/static/images/blog/es-th-2-7.png" alt="es-th-2-7" style="zoom:67%;clear:both; display: block; margin: auto" />

refresh 默认1s，可通过 index.refresh_interval 设置。refresh 流程大致如下：

1. in-memory buffer 中的文档写入到新的 segment 中，但 segment 是存储在文件系统的缓存中。此时文档已经可以被搜索到。
2. 最后清空 in-memory buffer。注意：Translog 没有被清空，为了将 segement 写到磁盘中。
3. 文档经过 refresh 后，segment 暂时写到文件系统缓存。这样避免了性能 IO 操作，又可以使文档搜索到， refresh 每秒执行一次，性能损耗较大，一般建议稍微延长这个 refresh 时间间隔，比如 5s。这样做到准实时就可以了。

<br/>

- **flush 过程**

每隔一段时间，如果 translog 变得越来越大，索引会被刷新（flush）；一个新的 translog 被创建，并且一个全量提交被执行<img src="https://img1.kiosk007.top/static/images/blog/es-th-2-9.png" alt="es-th-2-9" style="zoom:67%;clear:both; display: block; margin: auto" />



上个过程 segment 在文件系统缓存中，会有意外故障文档丢失，那么，为了保证文档不会丢失，需要将文档从文件缓存写入磁盘的过程就是 flush，写入磁盘后，清空 translog，具体过程如下：

1. 所有的内存缓冲区的文档被写入一个新的 Segment。
2. 缓冲区被清空。
3. 一个 Commit Point 被写入硬盘。
4. 文件系统缓存通过 fsync 被刷新（flush）。
5. 老的 translog 被删除。

<br/>



- **merge 过程**

由于自动刷新流程每秒会创建一个新的 Segment ，这样会导致短时间内的 Segment 数量暴增。而 Segment 数目太多会带来较大的麻烦。 每一个 Segment 都会消耗文件句柄、内存和cpu运行周期。更重要的是，每个搜索请求都必须轮流检查每个Segment ；所以 Segment 越多，搜索也就越慢。

Elasticsearch 通过在后台进行 Merge Segment 来解决这个问题，小的 Segment 被合并到 大的 Segment，然后这些大的Segment 再合并到更大的 Segment。

当索引的时候， refresh 操作会创建新的 Segment 并将Segment 打开供搜索使用，合并进程选择一小部分大小相似的段，并且在后台将他们合并到更大的段中。这并不会中断索引和搜索。

<img src="https://img1.kiosk007.top/static/images/blog/es-th-2-10.png" alt="es-th-2-10" style="zoom:67%;clear:both; display: block; margin: auto" />

合并大的段需要消耗大量的I/O和CPU资源，如果任其发展会影响搜索性能。Elasticsearch在默认情况下会对合并流程进行资源限制，所以搜索仍然 有足够的资源很好地执行。

<br/>

## 文档查询步骤顺序

下面是从主分片或者副本分片检索文档的步骤顺序：

<img src="https://img1.kiosk007.top/static/images/blog/es-th-2-21.png" alt="es-th-2-21" style="zoom:67%;clear:both; display: block; margin: auto" />

1. 客户端向 Node 1 发送获取请求。
2. 节点使用文档的 _id 来确定文档属于分片0，分片0的副本分片存在于所有的三个节点之上，请求会被路由到 Node 2 。
3. Node 2 将 文档返回给 Node 1，然后将文档返回给客户端。

<br/>

> 在处理读取请求时，协调结点在每次请求的时候都会通过轮询所有的副本分片来达到负载均衡。

<br/>

**文档读取过程中的详细过程**

> 所有的搜索系统一般都是两阶查询，第一阶段查询到匹配的 DocID，第二阶段再查询 DocID 对应的完整文档。

<img src="https://img1.kiosk007.top/static/images/blog/es-th-2-32.jpeg" alt="es-th-2-32" style="zoom:122%;clear:both; display: block; margin: auto" />

1. 在初始查询阶段时，查询会广播到索引中每一个分片拷贝（主分片或者副本分片），每个分片在本地执行搜索并构建一个匹配文档大小为 from + size 的优先队列。（搜索时会查询 Filesystem Cache ，但有部分数据还在 Memory Buffer,所以搜索是近实时的）。
2. 每个分片返回各自优先队列中所有文档 ID 和排序值给协调结点。
3. 协调节点辨别出哪些些文档需要被取回并向相关的分片提交多个 GET 请求，每个分片加载并丰富文档，所有文档取回后，协调结点返回结果给客户端。



参考：

[ES 详解-原理：ES原理之索引文档流程详解](https://pdai.tech/md/db/nosql-es/elasticsearch-y-th-3.html)
