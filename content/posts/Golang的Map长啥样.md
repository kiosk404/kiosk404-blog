---
title: Golang的Map长啥样
author: kiosk
math: true
tags:
  - golang
categories:
  - programming
date: 2022-11-12 11:06:00
---

Map 是每个编程语言必备的数据结构，就连lua这种胶水语言都支持其功能，map 的底层是一个 HashTable，Go 语言的map使用十分简易，但其内部实现不是简单的一个 HashTable。

<!--more-->

# 键值存储

提到键值存储，理所应当的就想到了哈希表，哈希表通常会有一堆的桶（Buckets）来存储键值对信息。

<img src="https://img1.kiosk007.top/static/images/blog/go_map1.png" alt="go_map1" style="zoom:50%;" />

## 哈希映射

键值对会被映射到桶里，那么这个桶怎么选择呢？先通过哈希函数（Hash function）将key处理一下，得到一个Hash值。在用这个 Hash 值在 m个桶里选择一个，桶的编号区间 0 到 m-1 。目前有两种方式比较常用

- **取模法**

将 hash 值取模， hash % m ，这样 键值对就会映射到一个桶里。

- **与运算法**

hash & (m-1) ，这里为了防止出现空桶情况，m必须是2的整数次幂。这样m的二进制表示一定只有一位为1，而 m - 1 的二进制一定是低于1的位数均为1。

比如 m=4，二进制表达式为 00000100，比如 m-1 的二进制表达式为 00000011

<br/>

## 哈希冲突

但是这里有个问题。当多个hash值映射到同一个 bucket 里怎么办？最常用的方法是 哈希拉链法，比如桶指向一个链表，当多个值映射到一个桶时，就追加到链表（还可以使用其他数据结构）尾部。

<img src="https://img1.kiosk007.top/static/images/blog/go_map2.png" alt="go_map2" style="zoom:80%;clear:both;margin:auto;display:block;" />



虽然有 哈希拉链法 这样的解决冲突的手段，但是适时地扩充 bucket 的个数也是十分必要的，当键值对过多时就需要扩充哈希桶，而 count / m  的比值就是作为是否要扩充的判断依据。这个比值就是 **负载因子 (Load Factor)**

<br/>

## 渐进式迁移

当迁移发生时，并不是直接将旧表中的内容直接迁移至新表中，而是渐进式的方式。一个指针指向新表，一个指针指向旧表。再记录迁移进度。

当哈希表在读写操作时，如果检测到当前处于扩容阶段，就完成一部分的键值对迁移任务。直到所有的键值对迁移到新桶，旧桶不再使用才算完成所有的迁移任务。

这样可以避免一次完全迁移导致的性能抖动。

<br/>

# Map 的底层结构

Go map 在语言底层是通过如下结构体来表征的，其位置在[go/src/cmd/compile/internal/types/type.go](https://link.zhihu.com/?target=https%3A//github.com/golang/go/blob/0bb6115dd6246c047335a75ce4b01a07c291befd/src/cmd/compile/internal/types/type.go%23L217)

```go
// Map contains Type fields specific to maps.
type Map struct {
    Key  *Type // Key type
    Elem *Type // Val (elem) type

    Bucket *Type // internal struct type representing a hash bucket
    Hmap   *Type // internal struct type representing the Hmap (map header object)
    Hiter  *Type // internal struct type representing hash iterator state
}
```

前两个字段分别为 key value，由于 go map 支持多种数据类型,，go 会在编译期推断其具体的数据类型。

Bucket 是 Hash 桶，Hmap 是底层使用的 HashTable 的元信息，比如当前哈希表中含有的元素数据，桶指针等，Hiter 是用于遍历 go map 的数据结构。


<br/>



## Hmap 

hmap 本质是一个指针，其指向一个 Hmap 的结构体。其具体的数据结构位于[src/runtime/map.go](https://link.zhihu.com/?target=https%3A//golang.org/src/runtime/map.go)，hmap 结构体描述了 Go Map 的关键信息。

```go
// A header for a Go map.
type hmap struct {
    // Note: the format of the hmap is also encoded in cmd/compile/internal/gc/reflect.go.
    // Make sure this stays in sync with the compiler's definition.
    count     int // # live cells == size of map.  Must be first (used by len() builtin)
    flags     uint8
    B         uint8  // log_2 of # of buckets (can hold up to loadFactor * 2^B items)
    noverflow uint16 // approximate number of overflow buckets; see incrnoverflow for details
    hash0     uint32 // hash seed

    buckets    unsafe.Pointer // array of 2^B Buckets. may be nil if count==0.
    oldbuckets unsafe.Pointer // previous bucket array of half the size, non-nil only when growing
    nevacuate  uintptr        // progress counter for evacuation (buckets less than this have been evacuated)

    extra *mapextra // optional fields
}

```

- count：键值对数目
- flag：用于标识当前map的状态，如正在被遍历，被写入
- B：桶的多少次幂（Golang 使用的是与运算方法）
- noverflow：溢出桶的数目（这个后面会说到）
- buckets：用于记录桶的位置
- oldbuckets：用于记录扩容阶段旧桶的位置
- nevacuate：用于记录将迁移的旧桶编号



<br/>

## bmap

bmap 是哈希桶的结构，由于 go map 的key和value可以是多种类型，因此哈希桶的数据类型也会随着 key 和 elem 数据类型的不同而不同，具体的数据类型是在编译期确定，所以其 bmap 在 go 的源码中没有被显示的定义出来，其具体的函数是在 [src/cmd/compile/internal/gc/reflect.go](https://link.zhihu.com/?target=https%3A//github.com/golang/go/blob/0bb6115dd6246c047335a75ce4b01a07c291befd/src/cmd/compile/internal/gc/reflect.go%23L83)

```go
type bmap struct {
    topbits  [8]uint8
    keys     [8]keytype
    elems    [8]elemtype
    overflow uintptr
}
```

一个哈希桶可以存放8个元素。

<img src="https://img1.kiosk007.top/static/images/blog/go_map3.png" alt="go_map3" style="zoom:50%;clear:both;margin:auto;display:block;" />

bmap 的 topbits 是哈希值的高8位。

为了让内存排列的更紧密，key全部放在一起，value全部放在一起。

keys 存放的全是键值对中的键，而elems中存放的全是键值对中的值。

最后的 overflow *bmap 指向一个溢出桶。



**溢出桶**

当一个 bmap 存放满之后，overflow 会指向一个溢出桶，溢出桶和bmap常规桶一样。

<img src="https://img1.kiosk007.top/static/images/blog/go_map4.png" alt="go_map4" style="zoom:50%;clear:both;margin:auto;display:block;" />

<br/>

# 总体结构

回到 Hmap。我们知道了实际的 KV 键值对是存在bmap中，而bmap不够时是会进行溢出，新的数据存放到溢出桶中。

<img src="https://img1.kiosk007.top/static/images/blog/go_map6.png" alt="go_map6" style="clear:both;margin:auto;display:block;zoom:50%" />

hmap 结构相当于 go map 的头, 它存储了哈希桶的内存地址, 哈希桶之间在内存中紧密连续存储, 彼此之间没有额外的 gap, 每个哈希桶最多存放 8 个 k/v 对, 冲突次数超过 8 时会存放到溢出桶中, 哈希桶可以跟随多个溢出桶, 呈现一种链式结构, 当 HashTable 的装载因子超过阈值(6.5) 后会触发哈希的扩容, 避免效率下降。

<br/>

# Go Map 的查找

当要根据 key 从 map 中查询对应的elem时。在go中有两种写法

方法一是 s := hashMap[key]

方法二是 s, ok := hashMap[key]

第一种方法在没有 key 时会返回零值，第二种方法中的 ok 在不存在 key 时会返回 false。其分别对应 mapaccess1 函数 和  mapaccess2 函数。

```go
// returns key, if not find, returns nil
func mapaccess1(t *maptype, h *hmap, key unsafe.Pointer) unsafe.Pointer 

// returns key and exist. if not find, returns nil, false
func mapaccess2(t *maptype, h *hmap, key unsafe.Pointer) (unsafe.Pointer, bool)

// returns both key and value. if not find, returns nil, nil
func mapaccessK(t *maptype, h *hmap, key unsafe.Pointer) (unsafe.Pointer, unsafe.Pointer)
```

给定一个 key 可以通过下面的操作找到它是否存在。

<img src="https://img1.kiosk007.top/static/images/blog/go_map7.webp" alt="go_map7" style="clear:both;margin:auto;display:block;zoom:70%" />

<br/>

# Go Map 的扩容

随着想 HashTable 中插入的元素越来越多，哈希桶的 cell 逐渐被填满，溢出桶的数量也越来越多，此时哈希冲突发生的频率也越来越高，HashTable的性能将不断下降。为了解决这个问题，此时需要对HashTable 进行扩容，而判断是否需要扩容就需要看装载因子 LoadFactor。



$$LoadFactor := \frac{Element.Length}{HashTable.Length}$$



其中 HashTable 的长度为 $\verb!2^B!$ ，其 LoadFactor 是 6.5 。超过这个数就会触发**翻倍扩容**。

分配的新桶是旧桶的2倍。如果旧桶选择是旧桶一个槽位，那么新桶会被分流到新桶的 2个槽位置。

如果LoadFactor没有超标，但是使用的溢出桶较多，就会触发**等量扩容**。如果常规桶小于  $\verb!2^15!$ ，而使用的溢出桶超过这个值，那么就会触发等量扩容，等量扩容应对的场景是出现了大量删除的 key 。

<br/>

# Go Map 的遍历

go map 的遍历本来是一件简单的事情，外层循环遍历所有的 Bucket，中层循环横向遍历所有的溢出桶，内层循环遍历 Bucket中的所有 k/v ，若没有扩容逻辑的话，确实这样即可，但是由于存在扩容逻辑，使得map遍历复杂性变得复杂。

由于 map 扩容逻辑的存在, map 的遍历是无序的。

而实际上即便我们在代码中硬编码一个固定的 map，没有做任何更改，仍然是不一样的结果，这是因为，go 设置了随机遍历起点，不仅起始 Bucket 是随机的, 对于 Bucket 中的起始 cell 也是随机的(这样做似乎是为了规避程序员故意使用这个 map 的顺序?), map 在迭代过程中, 需要检查 map 的状态, 如果 map 当前正处于扩容状态, 则需要检查遍历到的 Bucket, 若 Bucket 尚未搬迁, 则需要去该 Bucket 对应的 oldBucket 里遍历元素, 并且这里要注意因为 oldBucket 中的元素可能会分流到两个新 Bucket 中, 因此在遍历时只会取出会分流到当前 Bucket 的元素, 否则元素会被遍历两次。

<br/>
<br/>

# 参考：
{{< bilibili id=BV1Sp4y1U7dJ >}}






