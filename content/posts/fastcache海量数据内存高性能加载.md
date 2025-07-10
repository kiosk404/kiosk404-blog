---
title: fastcahce 海量数据内存高性能加载
author: kiosk
tags:
  - Golang
categories:
  - Programming
date: 2025-05-30 21:01:00
---

最近在日常的工作中遇到这样一个场景：我需要存储线上海量的拨测结果，内存里堆积着数以百万计的小对象，每个对象只有几百字节？当我满怀信心地以为 Go 的垃圾回收（GC）能帮你搞定一切时，却发现程序时不时地“卡顿”一下，CPU 飙升，而内存占用也远超预期。

深入分析后，我终于定位到了问题的症结所在：这个服务需要维护海量的业务元数据，每个元数据对象只有大约 **2000-5000 字节**，但总量却高达 **200 万个**。这些“小不点”们，在内存里正悄无声息地制造着一场“大麻烦”！

<!--more-->

粗略的算了一下，这么多小对象，每个就按2字节算，大概需要消耗 4GB 的内存，而我的机器是 16C 32G 的容器，尽管还有其他的消耗，但是不知为什么，经常内存常态消耗达到 20-23GB ，虽然还有其他要存储的数据，但是大头也就 4GB，为什么会这么耗内存呢？

 ![fastcache1](https://img1.kiosk007.top/static/images/blog/20250709225253-fastcache1.png)



于是我开始调研了一下 go 的内存管理。

# Go 原生内存管理

## Go 的内存分配

在 Go 语言中，当你通过某些操作创建数据时，内存会在两个主要区域进行分配：**栈（Stack）** 和 **堆（Heap）**。

1. **栈内存：**
   - **特点：** 快速、自动、受限。
   - **用途：** 主要用于存储函数调用的局部变量、函数参数以及函数返回地址。它的分配和释放是自动且高效的（“先进后出”的原则）。当函数执行完毕，其在栈上分配的所有内存都会被自动清理。
   - **限制：** 大小通常比较小（默认几 MB），不适合存储大量或生命周期长的变量。
2. **堆内存：**
   - **特点：** 慢速、灵活、无限（理论上）。
   - **用途：** 用于存储那些**生命周期不确定**或**大小在编译时无法确定**的数据，比如：
     - 使用 `new()` 或 `make()` 创建的对象（切片、映射、通道等）。
     - 使用 `&` 运算符获取地址的局部变量，如果这个变量在函数返回后仍被其他地方引用（即发生了**逃逸分析**）。
     - **字符串的实际内容数据**。
   - **管理：** 堆内存的分配和释放不是自动的，而是由 Go 运行时的 **垃圾回收器（GC）** 负责。GC 会定期扫描堆内存，找出不再被程序引用的对象，然后回收它们所占用的内存。

#### 堆内存分配会影响性能？

每次在堆上分配内存，Go 运行时都需要执行一系列操作：

- **寻找合适的内存块：** 遍历内部的空闲内存列表，找到一个足够大的连续内存块。
- **初始化：** 对找到的内存块进行清零或初始化。
- **更新元数据：** 记录这个内存块已被分配，并更新 GC 所需的各种数据结构。
- **GC 压力：** 最重要的是，**每个在堆上新分配的对象，都会成为 GC 的“追踪目标”**。GC 必须扫描这些对象的头部信息，判断它们是否仍被引用。对象越多，GC 的扫描和标记工作量越大，就越容易导致 CPU 占用率上升和程序暂停（即便 Go 的 GC 大部分是并发的，仍有短暂停顿）。



## 内存对齐 

最初我简单地把这些对象放在 `map[string]*MyTinyObject` 里（其实是 `sync.Map`），以为每个对象只占 2000-5000 字节。然而，Go 运行时在堆上分配内存时，为了**内存对齐**和管理需要，会给每个对象分配比实际大小更大的内存块，并附带额外的元数据。

假设每个对象平均 3500 字节。如果每个对象在 Go 堆上分配时因为内存对齐和元数据开销，实际占用 4KB（4096 字节，一个常见的对齐单位或分配粒度），那么： 200 万个对象 × 4096 字节/对象 = 8,192,000,000 字节 ≈ **8.2 GB**。 这仅仅是对象本身的内存，还没算键的开销和 `map` 结构自身的开销。如果用 `sync.Map`，加上键（假设平均 32 字节）和 `map` 条目开销（假设 32 字节）： 200 万个对象 × (4096 + 32 + 32) 字节/对象 ≈ **8.32 GB**。 这个量级，即便对于内存充足的服务器来说，也已经是不小的负担了。但是按 3500字节 * 200万  ≈ **6.5GB** ，这里就差了 **1.7GB** 多。

> 更多内存对齐相关的内容参考 ：[golang-的内存对齐](https://kiosk007.top/post/golang-%E7%9A%84%E5%86%85%E5%AD%98%E5%AF%B9%E9%BD%90/)

<br/>



## GC

200 万个独立的堆对象，对 Go 的垃圾回收器而言，简直就是一场“噩梦”。

- **扫描开销巨大：** GC 每次运行时，都必须扫描 200 万个对象的**全部头部信息和引用关系**。这会消耗大量的 CPU 资源。
- **频繁的 GC 周期：** 如果这些对象生命周期不一致，或者有部分更新，会频繁触发 GC。
- **长尾延迟：** 尽管 Go 的 GC 多数是并发的，但在某些关键阶段（如标记辅助、清扫辅助，以及 Stop-The-World 阶段），它依然需要占用应用程序 Goroutine 的 CPU 时间。对象数量越多，这些阶段持续的时间可能越长，导致用户可见的**长尾延迟**和服务响应抖动。在高并发场景下，这尤其致命。
- **内存碎片化：** 频繁分配和释放大量大小不一的对象，会导致堆内存出现碎片。这不仅降低内存利用率，还可能导致后续的大块内存分配失败，进一步加剧性能问题。



这是我要用到的结构体。

```go
type AddrQuality struct {
    AddrMapSet map[string]Quality // Go map 是引用类型
}

type Quality struct {
    Addr string           // string 是引用类型 (16 字节头部)
    Arg1 float32          // 4 字节
    Arg2 []*QualityInfo   // slice 头部：24 字节 (指向底层数组的指针、长度、容量)
    ...
}
```

虽然单个 `Quality` 对象看起来数据量不大，但它包含了**大量的引用类型（`map`、`string`、`slice`、`pointer`）**。每个引用类型都意味着**额外在堆上独立分配一个或多个对象**，且这些对象都需要 Go GC 的持续追踪。这才是导致内存膨胀和 GC 成为性能瓶颈的根本原因。

<br/>

# fastcache 简介

> fastcache 是一个线程安全并且支持大量数据存储的高性能缓存组件库。

- [fastcache](https://github.com/VictoriaMetrics/fastcache)

这是官方 `Github` 主页上的项目介绍，和 `fasthttp` 名字一样以 `fast` 打头，作者对项目代码的自信程度可见一斑。此外该库的核心代码非常轻量。



## 基准测试

官方给出了 `fastcache`, `bigcache`, 标准库 `map`, `sync.Map` 的基准测试比较结果。

```bash
GOMAXPROCS=4 go test github.com/VictoriaMetrics/fastcache -bench='Set|Get' -benchtime=10s
goos: linux
goarch: amd64
pkg: github.com/VictoriaMetrics/fastcache
BenchmarkBigCacheSet-4      	    2000	  10566656 ns/op	   6.20 MB/s	 4660369 B/op	       6 allocs/op
BenchmarkBigCacheGet-4      	    2000	   6902694 ns/op	   9.49 MB/s	  684169 B/op	  131076 allocs/op
BenchmarkBigCacheSetGet-4   	    1000	  17579118 ns/op	   7.46 MB/s	 5046744 B/op	  131083 allocs/op
BenchmarkCacheSet-4         	    5000	   3808874 ns/op	  17.21 MB/s	    1142 B/op	       2 allocs/op
BenchmarkCacheGet-4         	    5000	   3293849 ns/op	  19.90 MB/s	    1140 B/op	       2 allocs/op
BenchmarkCacheSetGet-4      	    2000	   8456061 ns/op	  15.50 MB/s	    2857 B/op	       5 allocs/op
BenchmarkStdMapSet-4        	    2000	  10559382 ns/op	   6.21 MB/s	  268413 B/op	   65537 allocs/op
BenchmarkStdMapGet-4        	    5000	   2687404 ns/op	  24.39 MB/s	    2558 B/op	      13 allocs/op
BenchmarkStdMapSetGet-4     	     100	 154641257 ns/op	   0.85 MB/s	  387405 B/op	   65558 allocs/op
BenchmarkSyncMapSet-4       	     500	  24703219 ns/op	   2.65 MB/s	 3426543 B/op	  262411 allocs/op
BenchmarkSyncMapGet-4       	    5000	   2265892 ns/op	  28.92 MB/s	    2545 B/op	      79 allocs/op
BenchmarkSyncMapSetGet-4    	    1000	  14595535 ns/op	   8.98 MB/s	 3417190 B/op	  262277 allocs/op
```



从测试的结果中可以看到:

- `fastcache` 在所有操作上都要比 `bigcache` 快
- `fastcache` 在 `只写 + 读写混合` 操作比标准库的 `map, sync.Map` 要快，`只读` 操作比后者要慢



## 示例

```go
package main

import (
	"fmt"

	"github.com/VictoriaMetrics/fastcache"
)

func main() {
	// 初始化一个大小为 32MB 的缓存
	cache := fastcache.New(32 * 1024 * 1024)

	key := []byte(`hello`)
	val := []byte(`world`)

	cache.Set(key, val)                      // 设置 K-V
	fmt.Println(cache.Has(key))              // true
	fmt.Println(cache.Has([]byte(`hello2`))) // false

	fmt.Printf("hello = %s\n", cache.Get(nil, key)) // hello= world
    
	cache.Del(key)
	fmt.Println(cache.Has(key)) // fasle
}
```

从示例代码可以看到，除了初始化时需要指定缓存的大小，组件提供的 API 就是常规的 “键值对” 语义操作，例如 `Get`, `Set`, `Del` 等。



# fastcache 为什么做到高效



## 摆脱GC束缚

FastCache 高效的基石，在于它对 Go 运行时内存分配的“反叛”。它没有让 Go 的垃圾回收器（GC）去追踪每一个缓存项，而是选择**自行管理一大块连续的内存**。

### 预分配大块内存，打造 GC 的“盲区”

传统 Go 程序中，每一次 `make` 或 `new` 都会导致一个新对象在堆上分配，并被 GC 纳入监管范围。对象数量越多，GC 扫描、标记和清理的工作量就越大。FastCache 巧妙地规避了这一点：



1. **集中式分配**：FastCache 在初始化时，会一次性向操作系统申请**一大块（或多块）连续的字节数组（`[]byte`）**。这些大块内存被称为 **arena** 或 **slab**。所有缓存的键值对数据，都会被**序列化**后，紧凑地填充到这些预分配的 `[]byte` 数组中。

2. **GC 的“视而不见”**：对 Go 的 GC 来说，它只需要管理 FastCache 自身少数几个核心的数据结构（比如指向这些大 `[]byte` 数组的指针、分片结构等），而不会深入这些 `[]byte` 数组的内部，去逐一扫描、标记你存储的每一个键值对。这就像 GC 面对的是一整本厚厚的书，而不是书里面密密麻麻的每一张独立卡片。这种机制极大地减少了 GC 的工作量，**显著降低了 GC 导致的停顿和 CPU 消耗**。

3. **极致内存紧凑**：因为数据是序列化后直接写入连续的字节流，它几乎消除了 Go 对象固有的**内存对齐填充、对象头部开销和指针引用**等额外负担。存储 100 字节的数据，它在 FastCache 的底层内存中就真的只占用 100 字节（或非常接近），**内存利用率因此飙升**。 



### 零 GC 内存分配的读写操作

一旦这些底层的大内存块被提前分配好，FastCache 在后续的 `Get` 和 `Set` 操作中，几乎不会产生新的堆内存分配。

- **高效读**：`Get` 操作直接从底层 `[]byte` 数组中定位到数据的偏移量和长度，然后返回对应的字节切片。
- **高效写**：`Set` 操作将键值对序列化成字节流后，直接写入预分配好的 `[]byte` 数组中。 这种 **“零分配”** 的模式，显著减少了 Go 运行时的调度和管理开销，使得数据操作速度极快。



### Ring Buffer && LRU 

FastCache 会采用一套精妙的策略来管理和重用底层内存。常见的实现方式包括：

- **循环缓冲区（Ring Buffer）**：当底层大内存块被写满时，FastCache 会从头部开始覆盖最老的数据。这种模式天然适合作为 LRU (最近最少使用) 淘汰策略的底层实现，因为它会自动淘汰最不常用的数据。
- **空闲列表/链表**：在某些更复杂的实现中，当缓存项被删除或过期时，其占据的空间可以被标记为“空闲”，并加入一个空闲空间列表，以便后续的新数据能够重用这些被释放的空间，避免内存碎片化。





## 分片并发

FastCache 通过**分片（Sharding）**机制，实现了卓越的并发性能：

### 逻辑分段，物理隔离



- FastCache 会将整个缓存空间逻辑上划分为多个独立的**“分片”（Shard）**。你可以配置分片的数量（通常是 2 的幂次方，例如 256 或 512）。
- 每个分片都是一个独立的单元，拥有自己的一套：
  - **底层内存块**（可能是一个独立的 Arena 或 Slab）。
  - **索引映射**（一个 Go 原生的 `map[string]ObjectLocation`，其中 `ObjectLocation` 存储的是数据在底层内存块中的偏移量和长度）。
  - **并发锁**（通常是一个 `sync.RWMutex` 或 `sync.Mutex`）。



### 哈希路由，独立加锁

- 当一个读写请求到来时，FastCache 会根据键（Key）的哈希值，快速地将该请求路由到对应的**唯一分片**。
- 然后，它只会锁定这个特定的分片进行操作。
- **关键优势：** 这就意味着，不同 Goroutine 针对不同键的并发读写请求，可以同时在**不同的分片上并行执行**，而不会相互阻塞！锁的粒度被大大缩小，显著提高了并发吞吐量。



### 降低锁竞争，提升吞吐量

- 如果只有一个全局锁，那么所有的缓存操作都将串行化执行，高并发下性能会急剧下降。
- 通过分片，大量的操作可以在不同的分片上并发进行，**降低了锁的竞争概率**，从而让 CPU 核心能够更充分地并行利用，极大地提升了整体吞吐量。



# 验证

## 内存消耗对比

下面验证一下 普通的 `map[string]*LogEntry` （存储指针） 和 `fastcache` (存储序列化 []byte)  的内存消耗对比。

**为 `LogEntry` 实现一个相对简单的 `MarshalBinary` 和 `UnmarshalBinary` 方法**，用于 `fastcache` 场景。这个方法会尝试将结构体内容紧凑地打包成 `[]byte`。

结构体定义如下：

```go
// --- 简洁的 LogEntry 结构体 ---
type LogEntry struct {
	Timestamp  int64  `json:"timestamp"`  // 8 字节
	Level      byte   `json:"level"`      // 1 字节
	Source     string `json:"source"`     // 字符串头 16 字节
	Message    string `json:"message"`    // 字符串头 16 字节
	UserID     uint32 `json:"user_id"`    // 4 字节
	IsCritical bool   `json:"is_critical"`// 1 字节
}

func (le *LogEntry) MarshalBinary() []byte {
	leBytes, err := json.Marshal(le)
	if err != nil {
		panic("marshal failed")
	}
	return leBytes
}

func (le *LogEntry) UnmarshalBinary(data []byte) error {
	if le == nil {
		return errors.New("LogEntry pointer is nil")
	}
	return json.Unmarshal(data, le)
}
```

在 64 位系统上，Go 编译器为了效率会进行内存对齐。我们来分析 `LogEntry` 的实际内存布局：

- **原始数据大小**：8 (Timestamp) + 1 (Level) + 16 (Source 头) + 16 (Message 头) + 4 (UserID) + 1 (IsCritical) = **46 字节**。

- **实际内存布局（64 位系统，考虑对齐和填充）：**

  1. `Timestamp` (int64): 占用 8 字节。
  2. `Level` (byte): 占用 1 字节。为了下一个字段 `Source` (string，头部 16 字节对齐) 的对齐，`Level` 后面会**填充 7 字节**。
  3. `Source` (string): 占用 16 字节 (字符串头，包含指针和长度)。
  4. `Message` (string): 占用 16 字节 (字符串头)。
  5. `UserID` (uint32): 占用 4 字节。
  6. `IsCritical` (bool): 占用 1 字节。为了整个结构体 8 字节对齐（因为有 int64 和 string 头部），`IsCritical` 后面会**填充 3 字节**。

  ```
  内存地址： 0-7       8    9-15    16-31     32-47     48-51     52     53-55
            +---------+---+-------+---------+---------+---------+---+-------+
  字段：     |Timestamp|Level|PADDING|Source(16B)|Message(16B)|UserID(4B)|IsCritical|PADDING|
  大小：       8B        1B     7B        16B         16B         4B       1B      3B
  ```

- **`LogEntry` 结构体实际在堆上的占用**：8 + 1 + 7 + 16 + 16 + 4 + 1 + 3 = **56 字节**。

  - **额外增加**：56 - 46 = **10 字节**的填充。

- **Go 对象头部开销**：当这个 `LogEntry` 对象在堆上被独立分配时（例如通过 `new(LogEntry)` 或作为 `map` 的值），Go 运行时还会为它添加 8 到 16 字节的**对象头部信息**（用于 GC 和类型信息）。所以，一个空的 `LogEntry` 实际在堆上可能占用 56 + 8 ~ 16 = **64 ~ 72 字节**。

- **字符串内容的额外开销**：最重要的是，`Source` 和 `Message` 这两个 **`string` 字段的实际字符数据**是**独立存储在堆上**的。如果 `Source` 平均 15 字节，`Message` 平均 50 字节：

  - `Source` 数据：15 字节。实际堆占用 `16 (头部) + 15 (数据) = 31 字节`。
  - `Message` 数据：50 字节。实际堆占用 `16 (头部) + 50 (数据) = 66 字节`。
  - 这意味着，一个 `LogEntry` 对象会额外导致 **2 个字符串对象**在堆上分配，总计约 `31 + 66 = 97 字节`。

**综上，一个逻辑上只有 46 字节数据，外加 65 字节字符串内容的 `LogEntry`，在内存中实际会分裂成至少 3 个独立对象，并总共占用约 `(64~72) + 31 + 66 = 161 ~ 169 字节`。**





定义2个辅助函数用于统计和展示内存统计



```go
// --- 辅助函数：内存统计 ---
func printMemUsage(msg string) {
	var m runtime.MemStats
	runtime.ReadMemStats(&m)
	fmt.Printf("--- %s ---\n", msg)
	fmt.Printf("Heap Alloc = %s\n", formatBytes(m.HeapAlloc)) // 当前堆上已分配并仍在使用的字节数
	fmt.Printf("Total Objects = %d\n", m.HeapObjects)         // 堆上对象的总数
	fmt.Printf("Sys Memory = %s\n", formatBytes(m.Sys))       // 操作系统为进程分配的总内存
	fmt.Printf("NumGC = %d\n\n", m.NumGC)                     // 完成的GC循环次数
}

func formatBytes(b uint64) string {
	const unit = 1024
	if b < unit {
		return fmt.Sprintf("%d B", b)
	}
	div, exp := uint64(unit), 0
	for n := b / unit; n >= unit; n /= unit {
		div *= unit
		exp++
	}
	return fmt.Sprintf("%.1f %cB", float64(b)/float64(div), "KMGTPE"[exp])
}
```



主函数如下：

```go
// --- 主函数 ---
func main() {
	const numObjects = 2_000_000 // 200万个对象

	// 1. 初始内存使用
	runtime.GC()                       // 强制GC，确保内存统计相对干净
	time.Sleep(time.Millisecond * 100) // 等待GC完成
	printMemUsage("Initial Memory Usage")

	// --- 场景一：使用普通的 map[string]*LogEntry ---
	fmt.Println("--- Storing objects in map[string]*LogEntry ---")
	objMap := make(map[string]*LogEntry, numObjects) // 预分配容量

	for i := 0; i < numObjects; i++ {
		key := "log_" + strconv.Itoa(i)
		entry := &LogEntry{
			Timestamp:  time.Now().UnixNano() + int64(i),
			Level:      byte(i % 5),
			Source:     "server_node_" + strconv.Itoa(i%100),
			Message:    fmt.Sprintf("User %d performed action X. Data: %s. Level %d", i, key, i%5), // 确保字符串长度变化
			UserID:     uint32(i),
			IsCritical: i%1000 == 0,
		}
		objMap[key] = entry
	}

	runtime.GC() // 强制GC
	time.Sleep(time.Millisecond * 100)
	printMemUsage("Memory after storing in map[string]*LogEntry")

	// 保持对 objMap 的引用，避免被GC过早回收
	_ = objMap["log_0"]

	// --- 清理 map，为 fastcache 场景腾出内存 ---
	fmt.Println("--- Cleaning map and running GC ---")
	objMap = nil // 解除引用，允许GC回收
	runtime.GC() // 强制GC
	time.Sleep(time.Millisecond * 100)
	printMemUsage("Memory after map cleanup")

	// --- 场景二：使用 fastcache ---
	fmt.Println("--- Storing objects in fastcache ---")
	// 估算 fastcache 容量：
	// 每个 LogEntry 序列化后：固定部分约 18 字节 + 2 * (字符串长度 + 4字节长度前缀)。
	// 假设 Source 平均 20 字节，Message 平均 80 字节。
	// 序列化后平均大小约为：18 + (20+4) + (80+4) = 18 + 24 + 84 = 126 字节。
	// 200万个对象 * 126字节/对象 = 252 MB。
	// 留出一些额外空间和 fastcache 内部索引的开销，例如 300 MB。
	cacheSize := 300 * 1024 * 1024 // 300 MB
	cache := fastcache.New(cacheSize)

	// 用于 fastcache.Get 的 dst 缓冲区，避免每次Get都重新分配
	// 估算单个序列化对象最大可能大小，例如 200字节
	getDstBuf := make([]byte, 200)

	for i := 0; i < numObjects; i++ {
		key := []byte("log_" + strconv.Itoa(i))
		entry := &LogEntry{
			Timestamp:  time.Now().UnixNano() + int64(i),
			Level:      byte(i % 5),
			Source:     "server_node_" + strconv.Itoa(i%100),
			Message:    fmt.Sprintf("User %d performed action X. Data: %s. Level %d", i, string(key), i%5),
			UserID:     uint32(i),
			IsCritical: i%1000 == 0,
		}

		serializedData := entry.MarshalBinary()
		cache.Set(key, serializedData)
	}

	runtime.GC() // 强制GC
	time.Sleep(time.Millisecond * 100)
	printMemUsage("Memory after storing in fastcache")

	// 验证 fastcache get (并保持引用)
	retrievedData := cache.Get(getDstBuf, []byte("log_0"))
	retrievedEntry := &LogEntry{}
	_ = retrievedEntry.UnmarshalBinary(retrievedData) // 实际应用需要检查错误

	fmt.Println("Demo finished. Observe the 'Heap Alloc' and 'Total Objects' differences.")
	fmt.Println("Note: fastcache's primary data is managed efficiently within its pre-allocated segments, significantly reducing Go GC's object count. Its internal index (map) still contributes to heap objects.")
}

```



输出如下：

```bash
➜  demo go run main.go                                   
--- Initial Memory Usage ---
Heap Alloc = 147.1 KB
Total Objects = 257
Sys Memory = 6.4 MB
NumGC = 1

--- Storing objects in map[string]*LogEntry ---
--- Memory after storing in map[string]*LogEntry ---
Heap Alloc = 412.0 MB
Total Objects = 8008429
Sys Memory = 451.7 MB
NumGC = 10

--- Cleaning map and running GC ---
--- Memory after map cleanup ---
Heap Alloc = 163.9 KB
Total Objects = 278
Sys Memory = 451.7 MB
NumGC = 11

--- Storing objects in fastcache ---
--- Memory after storing in fastcache ---
Heap Alloc = 72.4 MB
Total Objects = 10076
Sys Memory = 452.0 MB
NumGC = 78

Demo finished. Observe the 'Heap Alloc' and 'Total Objects' differences.
Note: fastcache's primary data is managed efficiently within its pre-allocated segments, significantly reducing Go GC's object count. Its internal index (map) still contributes to heap objects.

```





# 扩展阅读

- [golang本地缓存选型及原理总结](https://docs.google.com/presentation/d/1W2TIVAGFDe7ynjbhyGpc9SKB8Edg6ScUbwJyBWwyuUs/edit#slide=id.g120fbcd2d30_2_75)