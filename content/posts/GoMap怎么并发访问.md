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

Go语言原生的map类型在并发读写时会导致panic，这是Go设计团队在权衡性能和安全性后的决定。当多个goroutine同时执行写操作，或同时进行读写操作时，会触发并发检测机制导致程序崩溃。错误示例如下：

```
fatal error: concurrent map read and map write
```

根本原因在于Go的map实现中使用了**写检测机制**，当检测到并发写操作时，会主动抛出panic以避免数据竞争（Data Race）导致的不可预知行为。

<br/>

# 如何实现一个线程安全的 map 类型

最简单的方式必然是加锁访问这里不再赘述了。

虽然使用 **互斥锁** 或 **读写锁** 可以提供线程安全的 map，但是在大量并发读写的情况下，会出现非常激烈的锁竞争。

**锁是性能下降的万恶之源**，在高并发场景下，第一原则就是尽量减少锁的使用，一些单线程、单进程的应用（比如 redis 、nginx），基本上不需要使用锁去解决并发线程访问的问题。

<br/>



## 传统方法 (加锁)

```go
type SafeMap struct {
    mu sync.RWMutex
    m  map[string]int
}

func (s *SafeMap) Get(key string) int {
    s.mu.RLock()
    defer s.mu.RUnlock()
    return s.m[key]
}
```

**原理**：
读写锁（RWMutex）允许多个读操作同时进行，但写操作是排他的。读锁和写锁是分离的，读操作不会阻塞其他读操作，但会阻塞写操作。
**适用场景**：读多写少的场景（性能比Mutex提升40%+）。

<br/>

## 官方推荐 (sync.Map)

```bash
var sm sync.Map

// 存储
sm.Store("key", 1)

// 加载
value, ok := sm.Load("key")

// 原子操作
actual, loaded := sm.LoadOrStore("key", 2)
```

**原理**：
`sync.Map` 内部使用了**读写分离**的设计，通过两个map（read和dirty）实现无锁读操作。

- `read map`：存储只读数据，支持无锁访问。
- `dirty map`：存储新写入的数据，需要加锁访问。

当读操作频繁命中`read map`时，性能接近原生map；当需要写入时，`sync.Map`会将`read map`中的数据复制到`dirty map`中，并加锁更新。

**特性**：

- 适用于写少读多、键值相对稳定的场景。
- 自动优化并发性能。
- 类型不安全，需自行断言类型。



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

<br/>

## 分块无锁操作 (fastcache)

**原理**：
`fastcache` 是一个高性能的内存缓存库，基于**分块存储**和**无锁设计**实现。

- **分块存储**：将缓存数据分散到多个块（chunk）中，每个块独立管理，减少锁竞争。
- **无锁设计**：通过原子操作（atomic）实现并发安全，避免使用互斥锁。

**特点**：

- 高性能，适合高并发场景。
- 支持大容量缓存（GB级别）。
- 低GC压力，通过复用内存块减少内存分配。

```go
import "github.com/VictoriaMetrics/fastcache"

func main() {
    cache := fastcache.New(128 * 1024 * 1024) // 128MB缓存
	cache.Set([]byte("key"), []byte("value"))
	value := cache.Get(nil, []byte("key"))
}

```

<br/>

## 未来发展方向

Go团队正在探索的改进：

1. 泛型支持的并发容器。
2. 自动分片的内置map。
3. 无锁（lock-free）算法实现。
