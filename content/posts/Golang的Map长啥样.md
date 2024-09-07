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

Map 是每个编程语言必备的数据结构，就连 lua 这种胶水语言都支持其功能，map 的底层是一个 HashTable，Go 语言的map使用十分简易，但其内部实现不是简单的一个 HashTable。

<!--more-->

# 键值存储

提到键值存储，理所应当的就想到了哈希表，哈希表通常会有一堆的桶（Buckets）来存储键值对信息。

<img src="https://img1.kiosk007.top/static/images/blog/go_map1.png" alt="go_map1" style="zoom:50%;" />

## 哈希映射

键值对会被映射到桶里，那么这个桶怎么选择呢？先通过哈希函数（Hash function）将key处理一下，得到一个Hash值。在用这个 Hash 值在 `m`个桶里选择一个，桶的编号区间 0 到 `m-1` 。目前有两种方式比较常用

- **取模法**

将 hash 值取模， `hash % m `，这样 键值对就会映射到一个桶里。

- **与运算法**

`hash & (m-1)` ，这里为了防止出现空桶情况，`m` 必须是2的整数次幂。这样 `m` 的二进制表示一定只有一位为1，而 `m - 1` 的二进制一定是低于1的位数均为1。

比如 `m=4` ，二进制表达式为 00000100，比如 `m-1` 的二进制表达式为 00000011

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

前两个字段分别为 key  value，由于 go map 支持多种数据类型,，go 会在编译期推断其具体的数据类型。如 `map[string]int`

Bucket 是 Hash 桶，Hmap 是底层使用的 HashTable 的元信息，比如当前哈希表中含有的元素数据，桶指针等，Hiter 是用于遍历 go map 的数据结构。


<br/>



## Hmap 

hmap 本质是一个指针，其指向一个 Hmap 的结构体。其具体的数据结构位于[src/runtime/map.go](https://github.com/golang/go/blob/go1.16.2/src/runtime/map.go#L149)，hmap 结构体描述了 Go Map 的关键信息。

```go
// $GOROOT/src/runtime/map.go
type hmap struct {
    count     int     // 代表哈希表中的元素个数，调用 len(map) 时，返回的就是该字段值。
    flags     uint8   // 状态标志（是否处于正在写入的状态等）
    B         uint8   // buckets 的对数。如果 B = 5，则 buckets 数组的长度 = 2^B = 32，意味着有 32 个桶。
    noverflow uint16  // 溢出桶的数量
    hash0     uint32  // 生成 hash 的随机数种子

    // 指向 buckets 数组的指针，数组大小为 2^B，如果元素个数为 0，它为 nil。
    // buckets 数组中每个元素都是 bmap 结构
    buckets    unsafe.Pointer
    // 如果发生扩容，oldbuckets 是指向老的 buckets 数组的指针，
    // 老的 buckets 数组大小是新的 buckets 的 1/2，
    // 非扩容状态下，它为 nil。
    oldbuckets unsafe.Pointer
    nevacuate  uintptr        // 表示扩容进度，小于此地址的 buckets 代表已搬迁完成。

    extra *mapextra // 存储一些额外信息。
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

bmap 是哈希桶的结构，每个 bmap 底层都采用链表的结构，也就是我们常说的桶，一个 bmap 里面最多装 8 个key，这些 key 之所以会落入到同一个桶，是因为它们经过哈希计算后，哈希结果的低 8 位是相同的，在桶内，又会根据 Key 计算出 hash 的高 8 位来决定 key 到底落入桶的哪个位置，也就是 tophash 字段存储了 key 哈希值高 8 位。

<br/>

``` go
// $GOROOT/src/runtime/map.go
// A bucket for a Go map.
type bmap struct {
    // 长度为 8 的数组
    // 用来快速定位 key 是否在这个 bmap 中
    // 一个桶最多有8个槽位，如果key计算出来的tophash值在tophash数组中，则代表该 key 在这个桶中
    tophash [bucketCnt]uint8
}
```

<br/>

此外， tophash 还会存储一些状态值，用于表明当前桶单元的状态，这些状态值都是小于minTopHash 。

```go
// $GOROOT/src/runtime/map.go
emptyRest      = 0 // 表明此桶单元为空，且更高索引的单元也是空
emptyOne       = 1 // 表明此桶单元为空
evacuatedX     = 2 // 用于表示扩容迁移到新桶前半段区间
evacuatedY     = 3 // 用于表示扩容迁移到新桶后半段区间
evacuatedEmpty = 4 // 用于表示此单元已迁移
minTopHash     = 5 // key 的 tophash 值与桶状态值分割线值，小于此值的一定代表着桶单元的状态，大于此值的一定是 key 对应的 tophash 值

// tophash calculates the tophash value for hash.
func tophash(hash uintptr) uint8 {
    top := uint8(hash >> (goarch.PtrSize*8 - 8))
    if top < minTopHash {
        top += minTopHash
    }
    return top
}
```

<br/>

在实际运行的时候，由于 go map 的key和value可以是多种类型，因此哈希桶的数据类型也会随着 key 和 elem 数据类型的不同而不同，具体的数据类型是在编译期确定，所以其 bmap 在 go 的源码中没有被显示的定义出来，其具体的函数是在 [src/cmd/compile/internal/gc/reflect.go](https://github.com/golang/go/blob/go1.16.2/src/cmd/compile/internal/gc/reflect.go#L82) (go1.17 以前的版本)

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

为了让内存排列的更紧密，key 全部放在一起，value 全部放在一起。

keys 存放的全是键值对中的键，而elems中存放的全是键值对中的值。

最后的 overflow *bmap 指向一个溢出桶。



**溢出桶**

当一个 bmap 存放满之后，overflow 会指向一个溢出桶，溢出桶和bmap常规桶一样。

<img src="https://img1.kiosk007.top/static/images/blog/go_map4.png" alt="go_map4" style="zoom:50%;clear:both;margin:auto;display:block;" />

<br/>

## 负载因子

> 负载因子（load factor），用于衡量当前哈希表中空间的占用率的核心指标。主要目的是为了**平衡 bucket 的存储空间大小** 和 **查找元素时的性能高低** 。

Go1.17 之前其 默认的负载因子是 6.5 ，这是 Go 官方经过测试得出的数字，一起来看看官方的这份测试报告。

报告中共包含 4 个核心指标，如下所示：

| loadFactor | %overflow | bytes/entry | hitprobe | missprobe |
| ---------- | --------- | ----------- | -------- | --------- |
| 4.00       | 2.13      | 20.77       | 3.00     | 4.00      |
| 4.50       | 4.05      | 17.77       | 3.25     | 4.50      |
| 5.00       | 6.85      | 14.77       | 3.50     | 5.00      |
| 5.50       | 10.55     | 12.94       | 3.75     | 5.50      |
| 6.00       | 15.27     | 11.67       | 4.00     | 6.00      |
| 6.50       | 20.90     | 10.79       | 4.25     | 6.50      |
| 7.00       | 27.14     | 10.15       | 4.50     | 7.00      |
| 7.50       | 34.03     | 9.73        | 4.75     | 7.50      |
| 8.00       | 41.10     | 9.40        | 5.00     | 8.00      |

- loadFactor：装载因子
- %overflow：溢出率，有溢出 bucket 的百分比
- bytes/entry：平均没对 key/value 的开销字节数
- hitprobe：查找一个存在的 key 时，要查找的平均个数
- missprobe：查找一个不存在的 key 时，要查找的平均个数

<br/>

**装载因子越大，填入的元素越多，空间利用率就越高，但发生哈希冲突的几率就会变大。反之，装载因子越小，填入的元素越少，冲突的几率越小，但空间的浪费也会变多很多，而且还会提高扩容操作的次数。**

<br/>

也是根据这份测试结果，Go 官方取了一个相对适中的值，把 Go 中的 map 的负载因子硬编码为 6.5 。

这就意味着，Go1.17 以前的版本，**当 map 存储的元素个数大于或等于（6.5 * 桶个数）时，就会触发扩容行为。**

<br/>

Go1.18 ~ Go1.20 以后的逻辑有改变，map 的负载因子根据哈希表的性能动态调整。这里就不具体做介绍了



# 总体结构

回到 Hmap。我们知道了实际的 KV 键值对是存在bmap中，而bmap不够时是会进行溢出，新的数据存放到溢出桶中。

<img src="https://img1.kiosk007.top/static/images/blog/golang-map.drawio.png" alt="go_map6" style="clear:both;margin:auto;display:block;zoom:50%" />

<br/>

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

## map 遍历为什么是无序的？

- map 在遍历时，并不是从固定的 0 号 bucket 开始遍历，每次遍历，都会从一个随机值序号的 bucket , 再从其中的 cell 开始遍历。
- map 在遍历时，是按需遍历 bucket ，同时按需遍历 bucket 中和其 overflow bucket 中的 cell 。但在map 扩容后，会发生 key 的迁移，这就造成落在一个 bucket 中的 key ，随着迁移后，可能落到了其他 bucket 中，从这个角度看，只要扩容了，遍历 map 就不再有序了。



# Go Map 的扩容

随着想 HashTable 中插入的元素越来越多，哈希桶的 cell 逐渐被填满，溢出桶的数量也越来越多，此时哈希冲突发生的频率也越来越高，HashTable的性能将不断下降。为了解决这个问题，此时需要对HashTable 进行扩容，而判断是否需要扩容就需要看装载因子 LoadFactor。



$$LoadFactor := \frac{Element.Length}{HashTable.Length}$$



其中 HashTable 的长度为 $\verb!2^B!$ (不包含溢出桶)，其 LoadFactor 是 6.5 。超过这个数就会触发**翻倍扩容**。

分配的新桶是旧桶的2倍。如果旧桶选择是旧桶一个槽位，那么新桶会被分流到新桶的 2个槽位置。

如果LoadFactor没有超标，但是使用的溢出桶较多，就会触发**等量扩容**。如果常规桶小于  $\verb!2^15!$ ，而使用的溢出桶超过这个值，那么就会触发等量扩容，等量扩容应对的场景是出现了大量删除的 key 。

<br/>

## 扩容规则

map 的扩容时使用的是**渐进式扩容**。

由于 map 扩容需要将原有的 key/value 重新搬到新的内存地址，一次性搬运大量的 key-value 时势必会造成比较大的延迟，因此 Go map 的扩容是渐进式的，原有的 Key 并不会一次性搬迁完毕，每次最多只会搬迁 2 个 bucket，只有在插入或删除、修改 key 的时候，都会尝试进行搬迁 bucket 的工作，先检查 oldbuckets 是否搬迁完毕，具体来说就是检查 oldbuckets 是否为 nil 



<br/>

# Go Map 的缩容

在 Go 底层源码 src/runtime/map.go 中，扩缩容的处理方法是 grow 为前缀的方法来处理的。

是否进行缩容的条件是 溢出桶（noverflow）的数量 >= 32768 (1<<15) 。

而缩容本质上是等效扩容。

由此可以得出结论：map 的**扩缩容的主要区别在于 hmap.B 的容量大小改变**。而缩容由于 hmap.B 压根没变，内存空间的占用也是没有变化的。

<br/>

<font color='red'>这种方式其实是存在运行隐患的</font> :tired_face:，也就是**导致在删除元素时，并不会释放内存，使得分配的总内存不断增加**。如果一个不小心，拿 map 来做大 key/value 的存储，也不注意管理，很容易就内存爆了。

也就是 Go 语言的 map 目前实现的是 ”伪缩容“，仅针对溢出桶过多的情况。若是触发缩容，hash 数组的占用的内存大小不变。

## 为什么不支持缩容？

下述内容会主要基于如下两个 issues 和 proposal 来分析：

1. 《[runtime: shrink map as elements are deleted](https://github.com/golang/go/issues/20135)》
2. 《[proposal: runtime: add way to clear and reuse a map’s working storage](https://github.com/golang/go/issues/45328)》

目前 map 的缩容处理起来比较棘手，最早的 issues 是 2016 年提出的，也有人提过一些提案，但都因为种种原因被拒绝了。

简单来讲，就是没有找到一个很好的方法实现，存在明确的实现成本问题，没法很方便的 ”告诉“ Go 运行时，我要：

1. 记得保留存储空间，我要立即重用 map。
2. 赶紧释放存储空间，map 从现在开始会小很多。

抽象来看症结是：**需要保证增长结果在下一个开始之前完成**，此处的增长指的是 ”从小到大，从一个大小到相同大小，从大到小“ 的复杂过程。

<br/>

## 那怎么怎么避免内存泄漏呢？

- 如果删除的元素是值类型，如int，float，bool，string以及数组和struct，map的内存不会自动释放

- 如果删除的元素是引用类型，如指针，slice，map，chan等，map的内存会自动释放，但释放的内存是子元素应用类型的内存占用

- 将map设置为nil后，内存被回收





# Go Map 的遍历

go map 的遍历本来是一件简单的事情，外层循环遍历所有的 Bucket，中层循环横向遍历所有的溢出桶，内层循环遍历 Bucket中的所有 k/v ，若没有扩容逻辑的话，确实这样即可，但是由于存在扩容逻辑，使得map遍历复杂性变得复杂。

由于 map 扩容逻辑的存在, map 的遍历是无序的。

而实际上即便我们在代码中硬编码一个固定的 map，没有做任何更改，仍然是不一样的结果，这是因为，go 设置了随机遍历起点，不仅起始 Bucket 是随机的, 对于 Bucket 中的起始 cell 也是随机的(这样做似乎是为了规避程序员故意使用这个 map 的顺序?), map 在迭代过程中, 需要检查 map 的状态, 如果 map 当前正处于扩容状态, 则需要检查遍历到的 Bucket, 若 Bucket 尚未搬迁, 则需要去该 Bucket 对应的 oldBucket 里遍历元素, 并且这里要注意因为 oldBucket 中的元素可能会分流到两个新 Bucket 中, 因此在遍历时只会取出会分流到当前 Bucket 的元素, 否则元素会被遍历两次。

<br/>
<br/>

# 参考：
{{< bilibili id=BV1Sp4y1U7dJ >}}






