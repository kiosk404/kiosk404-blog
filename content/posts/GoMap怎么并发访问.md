---
title: Golang Map怎么并发访问
author: kiosk
math: true
tags:
  - golang
categories:
  - programming
date: 2023-07-08 11:06:00
---

Map 是每门编程语言的最基础的数据结构，这种数据结构的实现就是 key-value之间的映射关系，每个key都有一个唯一的索引值，通过索引值可以很快的找到对应的值。

Map 也是程序中最常见的数据结构，我们的项目中有大量的数据需要加载到内存中，而且需要高频访问，无疑 map 就是最好的选择，但是众所周知，Map是不支持并发访问的。

如下：

```go

func main() {
    var m = make(map[string]int,10) // 初始化一个map
    go func() {
        for {
            m["abc"] = 1 //设置key
        }
    }()

    go func() {
        for {
            _ = m["def"] //访问这个map
        }
    }()
    select {}
}
```

虽然这段代码的不同goroutine 是各自在操作不同的元素，但是运行时检测到同时对 map 对象有并发访问，就会直接 panic。



<img src="https://img1.kiosk007.top/static/images/blog/82fb958bb73128cf8afc438de4acc862.webp" alt="go_map6" style="clear:both;margin:auto;display:block;zoom:120%" />



<br/>

# 如何实现一个线程安全的 map 类型

最简单的方式必然是加锁访问这里不再赘述了。

虽然使用 **互斥锁** 或 **读写锁** 可以提供线程安全的 map，但是在大量并发读写的情况下，会出现非常激烈的锁竞争。

**锁是性能下降的万恶之源**，在高并发场景下，第一原则就是尽量减少锁的使用，一些单线程、单进程的应用（比如 redis 、nginx），基本上不需要使用锁去解决并发线程访问的问题。

<br/>

## 分片加锁 (concurrent-map)

一个减少锁的粒度的常用方法就是分片（shard），将一把锁分成几把锁，每个锁分别控制一个分片， Go 比较知名的分片并发 map 的实现是 [orcaman/concurrent-map](https://github.com/orcaman/concurrent-map)

<br/>

它默认采用 32 个分片，GetShard 是一个关键的方法，可以根据 key 算出分片的索引。

```go


    var SHARD_COUNT = 32
  
    // 分成SHARD_COUNT个分片的map
  type ConcurrentMap []*ConcurrentMapShared
  
  // 通过RWMutex保护的线程安全的分片，包含一个map
  type ConcurrentMapShared struct {
    items        map[string]interface{}
    sync.RWMutex // Read Write mutex, guards access to internal map.
  }
  
  // 创建并发map
  func New() ConcurrentMap {
    m := make(ConcurrentMap, SHARD_COUNT)
    for i := 0; i < SHARD_COUNT; i++ {
      m[i] = &ConcurrentMapShared{items: make(map[string]interface{})}
    }
    return m
  }
  

  // 根据key计算分片索引
  func (m ConcurrentMap) GetShard(key string) *ConcurrentMapShared {
    return m[uint(fnv32(key))%uint(SHARD_COUNT)]
  }
```



