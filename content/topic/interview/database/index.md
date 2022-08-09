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
  - [Redis使用]()
    - [Redis 有什么操作会导致其变慢](/topic/interview/database/#redis-有什么操作会导致其变慢)
    - 

- [Elasticsearch](/topic/interview/database/#elasticsearch)



## Redis

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

这里的单线程是指 Redis 的网络 I/O 是由一个线程来完成的，这即是 Redis 对外提供键值存储服务的主要流程。其余的功能，如持久化、一部删除、集群同步都是额外的线程执行

主要原因也是因为多线程会存在加锁问题，多线程并发访问问题。

但是 Redis 采用了 I/O 多路复用技术，epoll 基于事件回调机制，针对不同的场景的发生调用相应的处理函数。这样Redis就不需要一直轮询时间的发生。

> Redis 6.0 其实也支持了多线程模型，当然针对的是客户端的读写是并行的，每个命令的执行依旧是单线程

<br/>

#### zset 的底层实现

zset 是有序字典，自动去重的集合数据类型，其底层的实现为字典（dict）+ 跳表（skiplist）。

<br/>

#### Redis 的 key 淘汰原理



<br/>

#### Redis的持久化机制



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

<br/>







## Elasticsearch



