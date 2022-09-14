---
title: DataBase
subtitle: A database is an organized collection of structured information
disallow: true
lightgallery: true
comment: false
date: 2022-08-08 23:41:26
---



> Based on Redis



- [Redis](/topic/interview/database/#redis)
  - [Redis 基础](/topic/interview/database/#redis-基础)
    - [Redis 底层有哪些数据结构](/topic/interview/database/#redis-底层有哪些数据结构)
    - [rehash 的过程](/topic/interview/database/#rehash-的过程)
    - [Redis 为什么是一个单线程模型](/topic/interview/database/#redis-为什么是一个单线程的模型)
    - [zset的底层实现](/topic/interview/database/#zset-的底层实现)
    - [Redis的Key过期机制和淘汰原理](/topic/interview/database/#redis-的-key-过期机制和淘汰原理)
    - [Redis的持久化机制](/topic/interview/database/#redis的持久化机制)
    - [Redis 如何实现高可用](/topic/interview/database/#redis-如何实现高可用)
    - [Redis 分布式锁](topic/interview/database/#redis-分布式锁)
  - [Redis AOF 日志文件太大怎么办](/topic/interview/database/#redis-aof-日志文件太大怎么办)
  - [Redis使用](/topic/interview/database/#redis-使用)
    - [Redis 有什么操作会导致其变慢](/topic/interview/database/#redis-有什么操作会导致其变慢)
    - [Redis 在时间序场景的应用](/topic/interview/database/#redis-在时间序场景的应用)
- [Elasticsearch](/topic/interview/database/#elasticsearch)
  - [什么是倒排索引](/topic/interview/database/#什么是倒排索引)
  - [文档索引步骤顺序](/topic/interview/database/#文档索引步骤顺序)
- [MySQL](/topic/interview/database/#mysql)
  - [myisam和innodb的区别](/topic/interview/database/#myisam-和-innodb-的区别)
  - [MySQL的索引有哪些](/topic/interview/database/#mysql-的索引有哪些)
  - [MySQL的锁类型有哪些](/topic/interview/database/#mysql的锁的类型有哪些呢)
  - [事务的基本特性和隔离级别](/topic/interview/database/#事务的基本特性和隔离级别)
  - [MySQL主从同步是如何实现的](/topic/interview/database/#mysql-主从同步是如何实现的)

<br/>

## Redis

[20道Redis经典面试题](https://zhuanlan.zhihu.com/p/427496556)

### Redis 基础

#### Redis 底层有哪些数据结构

Redis 的底层的数据结构有 6 种，简单动态字符串、双向链表、整数数组、压缩列表、哈希表、跳表。

详见：[redis 为什么这么快](https://kiosk007.top/post/redis%E4%B8%BA%E4%BB%80%E4%B9%88%E8%BF%99%E4%B9%88%E5%BF%AB/)

<br/>

#### rehash 的过程

首先因为大量的数据写入会造成 哈希冲突，所以需要增大哈希桶，进行 rehash 过程。

rehash 的时机：当写入元素过多达到一定比例，同时，Redis 目前没有执行 RDB 或者 AOF 操作时。

Redis 采取的是 **渐进式 rehash** 。

Redis 有两个全局 hash 表，当其中一个需要扩容时，将另一个哈希表扩大为当前需扩容哈希表的一倍。

在处理请求时会顺带进行数据迁移，后台也会启动周期性的任务进行迁移。

<br/>

#### Redis 为什么是一个单线程的模型

这里的单线程是指 Redis 的网络 I/O 是由一个线程来完成的，这即是 Redis 对外提供键值存储服务的主要流程。其余的功能，如持久化、异步删除、集群同步都是额外的线程执行

主要原因也是因为多线程会存在加锁问题，多线程并发访问问题。

但是 Redis 采用了 I/O 多路复用技术，epoll 基于事件回调机制，针对不同的场景的发生调用相应的处理函数。这样Redis就不需要一直轮询时间的发生。

> Redis 6.0 其实也支持了多线程模型，当然针对的是客户端的读写是并行的，每个命令的执行依旧是单线程

<br/>

#### zset 的底层实现

zset 是有序字典，自动去重的集合数据类型，其底层的实现为字典（dict）+ 跳表（skiplist）。

<br/>

#### Redis 的 key 过期机制和淘汰原理

Key 过期策略：

1. 惰性删除：在取出键时才对键进行过期检查，如果发现过期了就会被删除
2. 定期删除：定期每隔一段时间执行一次删除过期键的操作。

内存淘汰机制：

1. volatile-lru：从已设置过期时间的数据集中挑选最少使用的数据淘汰
2. volatile-ttl：从已设置过期时间的数据集中挑选将要过期的进行数据淘汰
3. volatile-random：从已设置过期时间的数据集中任意选择数据淘汰
4. valatile-lfu：从已设置过期时间的数据集挑选使用频率最低的数据淘汰
5. allkeys-lru：从数据（server.db[i].dict）中挑选最近最少使用的数据淘汰
6. allkeys-lfu：从数据（server.db[i].dict）中挑选使用频率最低的数据淘汰
7. allkeys-random：从数据（server.db[i].dict）中随意淘汰
8. no-enviction（驱逐）：禁止驱逐数据，这也是默认策略。意思是当内存不足以容纳新入数据时，新写入操作就会报错，请求可以继续进行，线上任务也不能持续进行，采用no-enviction策略可以保证数据不被丢失。

详见：[https://www.cnblogs.com/pinxiong/p/13288087.html](https://www.cnblogs.com/pinxiong/p/13288087.html)

<br/>

#### Redis的持久化机制

AOF ：通过靠保存 Redis 服务器所执行命令日志的方式

RDB：通过内存快照的机制把内存中的数据集写入磁盘，也就是 Snapshot 快照（数据库中所有键值对数据）

详见：[https://kiosk007.top/post/redis%E6%8C%81%E4%B9%85%E5%8C%96%E6%9C%BA%E5%88%B6/](https://kiosk007.top/post/redis%E6%8C%81%E4%B9%85%E5%8C%96%E6%9C%BA%E5%88%B6/)

<br/>

#### Redis 如何实现高可用

Redis 实现高可用有三种部署模式：**主从模式，哨兵模式，集群模式**。

- **主从模式**：Redis部署了多台机器，有主节点，负责读写操作，有从节点，只负责读操作。从节点的数据来自主节点，实现原理就是**主从复制机制**
- **哨兵模式**：主从模式中，一旦主节点由于故障不能提供服务，需要人工将从节点晋升为主节点，同时还要通知应用方更新主节点地址。而哨兵模式就是为了解决该问题。
- **集群模式**：哨兵模式基于主从模式，实现读写分离，它还可以自动切换，系统可用性更高。但是它每个节点存储的数据是一样的，浪费内存，并且不好在线扩容。因此，Cluster集群应运而生，它在Redis3.0加入的，实现了Redis的**分布式存储**。对数据进行分片，也就是说**每台Redis节点上存储不同的内容**，来解决在线扩容的问题。并且，它也提供复制和故障转移的功能。

<br/>

#### redis 分布式锁



<br/>

#### redis AOF 日志文件太大怎么办

AOF 重写

<br/>





### Redis 使用

#### Redis 有什么操作会导致其变慢

Redis 处理变慢主要原因有两个。

1. Redis 本身的值的读写是单线程，任意一个请求阻塞都会导致 Redis 请求变慢

2. 并发量特别大时，单线程读写客户端 I/O 数据存在瓶颈。



可能导致慢的实际操作：

1. 大 Key 操作，写入删除一个大Key会消耗更多的时间；
2. HGETALL 等 O(N) 操作：
3. 大量的 key 集中过期：
4. 数据持久化，如 AOF 的 always 机制，主从全量同步生成 RDB 等。



针对问题1，除了需要开发人员从代码角度尽量避免大key，另一方面 Redis 在 4.0 推出 lazy-free 机制，将 大key 的释放内存的耗时操作都放到了异步线程中执行，避免对主线程的影响。

针对问题2，由于单个实例可能会造成在持久化时压力过大，可以考虑多实例。

<br/>

#### Redis 在时间序场景的应用

场景：已有大量的pcap格式的网络数据包，当计算出网络数据流的重传数据包，则表示当前网络中出现了重传，并且可能出现了丢包。需要计算在时间序中哪个时间段出现的丢包较为频繁。丢包是否是单独某条连接的问题。



使用 zset 和 hash 表。

zset 可以获取一个时间范围的丢包信息。而hash表可以获取到某个具体时间的丢包信息。

```bash
// HASH 表
HGET task_id:traffic_stream 202208030905
"23"

HMGET task_id:traffic_stream 202208030905 202208030907 202208030908
1）"23"
2) "30"
3) "0"

// ZSET
ZRANGEBYSCORE task_id:traffic_stream 202208030905 202208030908
1）"23"
2) "30"
3) "0"
```

另外也可以基于 RedisTimeSeries 模块保存时间序列数据

详情：[https://time.geekbang.org/column/article/282478](https://time.geekbang.org/column/article/282478)

<br/>



## Elasticsearch

### 什么是倒排索引

倒排索引包含两个部分：单词词典和倒排表。

- 单词词典（Term Dictionary）：记录所有文档的单词，记录单词到倒排列表的关联关系（单词词典一般比较大，通过 B+ 树或哈希拉链法实现，以满足高性能的插入与查询）
- 倒排表（Posting List）：记录了单词对应的文档结合，由倒排索引组成。
  - 文档ID
  - 词频 TF - 该单词在文档中分词的位置。用于语句搜索
  - 位置（Position）- 单词在文档中分词的位置，用于语句搜索
  - 偏移（Offset）- 记录单词的开始结束位置，实现高亮显示。

详见：[https://kiosk007.top/post/elasticsearch/](https://kiosk007.top/post/elasticsearch/)

<br/>

### 文档索引步骤顺序

1. 客户端向某节点1 发送新建、索引、或者删除请求
2. 节点1 根据文档 _id 判断文档所属分片0，而分片0 在节点2上，请求被转发至Node2。
3. Node2 执行请求，成功后，请求并行同步到Node1 和 Node3 的副本分片。一旦所有的副本分片都报告成功，Node2 回复 协调节点，协调节点向客户端报告成功。

<br/>



## MySQL

[MySQL常见面试题](https://zhuanlan.zhihu.com/p/222958908)

### myisam 和 innodb 的区别？

myisam 不支持事务、行级锁还有外键，所以一般用于有大量查询少量插入的场景。

innodb 是基于聚簇索引建立的，innodb 支持 事务还有外键。

<br/>

### MySQL 的索引有哪些？

按照数据结构来说，索引包含 B+树索引 和 Hash索引

<br/>

### MySQL的锁的类型有哪些呢？

mysql锁分为**共享锁**和**排他锁**，也叫做读锁和写锁。

读锁是共享的，可以通过lock in share mode实现，这时候只能读不能写。

写锁是排他的，它会阻塞其他的写锁和读锁。从颗粒度来区分，可以分为**表锁**和**行锁**两种。

表锁会锁定整张表并且阻塞其他用户对该表的所有读写操作，比如alter修改表结构的时候会锁表。

行锁又可以分为**乐观锁**和**悲观锁**，悲观锁可以通过for update实现，乐观锁则通过版本号实现。

<br/>

### 事务的基本特性和隔离级别

事务基本特性ACID分别是：

**原子性**：一个事务中的操作要么全部成功，要么全部失败

**一致性**：数据库总是从一个一致性的状态转换到另一个一致性的状态。（比如A给B转帐100元，sql 执行失败，A不会损失100，B也不会多出100）

**隔离性**：一个事务的修改在最终提交之前，对其他事务不可见

**持久性**：事务一旦提交，所做的修改就会永久的保存到数据库中。

隔离性有4个隔离级别，分别是：

- **read uncommit** 读未提交，可能会读到其他事务未提交的数据，也叫做脏读。
- **read commit** 读已提交，同一次事务中两次读取结果不一致，叫做不可重复读。
- **repeatable read** 可重复读，MySQL的默认级别，就是每次读取结果都一样，但是会产生幻读。
- **serializable** 串行，一般不会使用，他会给每一行读数据加锁，会导致大量超时问题。

<br/>

### MySQL 主从同步是如何实现的

主从同步原理：

1. master 提交完事务后，写入 binlog
2. master 创建dump线程，推送binglog 到 slave
3. slave 再开启一个IO线程读取同步过来的master的binlog，记录到relay log 中继日志中
4. slave 再开启一个 SQL 线程读取 relay log 事件并在 slave 执行，完成同步
5. slave 记录自己的binlog

由于mysql默认的复制方式是异步的，主库把日志发送给从库后不关心从库是否已经处理，这样会产生一个问题就是假设主库挂了，从库处理失败了，这时候从库升为主库后，日志就丢失了。由此产生两个概念。

**全同步复制**

主库写入binlog后强制同步日志到从库，所有的从库都执行完成后才返回给客户端，但是很显然这个方式的话性能会受到严重影响。

**半同步复制**

和全同步不同的是，半同步复制的逻辑是这样，从库写入日志成功后返回ACK确认给主库，主库收到至少一个从库的确认就认为写操作完成。
