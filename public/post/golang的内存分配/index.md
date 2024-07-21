# Golang的内存分配


程序中的数据和变量都会被分配到程序所在的虚拟内存当中，而内存空间包含两个重要区域：栈区（Stack）和堆区（Heap）。

<!--more-->

函数调用的参数、返回值以及局部变量大部分都会被分配到栈上，这部分内存会由编译器进行管理。不同的编程语言使用不同的方法管理堆区的内存，C++ 等编程语言需要程序员自己申请和释放内存，而 Golang 及 Java 等编程语言是程序员和编译器共同管理，堆中的对象由分配其分配并由垃圾回收器回收。



> golang 1.17 之后调用参数是分配到寄存器上的，这块我在 ebpf 程序追踪 Golang 的调用时踩过坑。



Golang 的内存分配其借鉴了 tcmalloc 的思想，尽量减少在多线程模型下，锁的竞争开销，来提高分配内存的效率。

<br/>

# TCMalloc

tcmalloc，其实就是 thread cache malloc 的缩写。

<img src="https://img1.kiosk007.top/static/images/blog/tcmalloc.jpg" alt="tcmalloc" style="zoom:77%;margin:auto;display:block;clear:both;" />

tcmalloc 内存分配分为 **ThreadCache**（线程缓存）、**CentralCache**（中心缓存）、**PageHeap**（页堆） 三个层次。

- ThreadCache 是一个线程的缓存，分配时不需要加锁，速度比较快。ThreadCache对于每一个size class 维护一个单独的FreeList，缓存还没有分配的空闲对象。
- CentralCache 也同样为一个size class 维护一个 FreeList，但是是所有线程公用的，分配时需要加锁。
- CentralCache 中的内存不够用时，会从 PageHeap 中申请，CentralCache 从 PageHeap 中申请的内存，可能来自于 PageHeap 的缓存，也可能是PageHeap 从操作系统中申请的新的内存。PageHeap 内存，对于128个Page以内的span，会使用一个链表来缓存，超过128 个 Page 的span，则存储于一个有序的 set 中。



> size class 是指将 8Bytes 、 16Bytes、32Bytes ... 32KB 都写成一个 size class

<br/>

其优点1. 通过线程缓存减少了锁的争用。2. 减少了系统调用和上下文切换。

所以 **TCMalloc 在多线程小内存的分配特别友好**，下图是 TCMalloc 作者给出的性能测试数据，看到线程数越多，二者的速度差距越大，所以当应用场景设计大量的并发线程时，换成 TCMalloc 库更有优势！

<img src="https://img1.kiosk007.top/static/images/blog/tcmalloc1.webp" alt="tcmalloc1" style="zoom:107%;margin:auto;display:block;clear:both;" />



<br/>

# Golang 的内存管理

Go 语言的 runtime 将堆内存划分成了一个一个的 arena ，而 arena 的起始地址被定义为 arenaBaseOffset ，在 amd64 的 Linux 环境下，每个 arena 的大小是 64MB，每个arena 包含8192 个 page，所以每个 page 的大小为 8KB。

<br/>

<img src="https://img1.kiosk007.top/static/images/blog/memory3.png" alt="memory3" style="zoom:50%;margin:auto;display:block;clear:both;" />

<br/>

因为程序在运行过程中申请使用的内存有大有小，而且在使用过程中不可避免的会产生很多碎片，为降低碎片内存化给程序性能造成的不良影响，**Go 的内存分配采用了和 TCMalloc 类似的算法**。

<br/>

其基本思路是按照一组预置的大小规格把内存页划分成块。然后将不同规格的**内存块**放入对应的空闲链表中。

程序申请内存时，分配器根据要申请的内存大小找到最匹配的规格。

Go1.16 的runtime 给出了67 种大小规格，最小的 8B ，最大32KB

<img src="https://img1.kiosk007.top/static/images/blog/memory4.png" alt="memory4" style="zoom:50%;margin:auto;display:block;clear:both;" />

<br/>

下面是 arena 、span 、page、内存块的 概念合集（span 的概念下面会提到）

<img src="https://img1.kiosk007.top/static/images/blog/memory5.png" alt="memory5" style="zoom:50%;margin:auto;display:block;clear:both;" />

<br/>

Go 语言的内存分配其包含内存管理单元、线程缓存、中心缓存和页堆几个重要组件，其对应的数据结构分别是 `mspan`、`mcache`、`mcentral` 和 `mheap` 。

<br/>

<img src="https://img1.kiosk007.top/static/images/blog/memory1.png" alt="memory1" style="zoom:50%;margin:auto;display:block;clear:both;" />

<br/>

所有 Go 语言程序都会在启动时初始化如上图所示的内存布局，每一个处理器 （P）都会分配一个线程缓存 `mcache` 用于处理微对象和小对象的分配，他们会持有内存管理单元 `mspan`。

每个类型的内存管理单元都会管理特定大小的对象（8B、16B、32B ... 32KB）当内存管理单元中不存在空闲对象时，他们会从 `mcentral` 中获取新的内存单元，中心缓存属于全局的堆结构体 `mheap` ，`mheap` 会从操作系统中申请内存。

<br/>

## 内存管理单元 mspan

mspan 是一个内存管理的基本单元。是一段连续的内存空间。

每个 mspan 都对应一个大小等级，小对象类型的堆对象会根据其大小分配到相应设定好的大小等级的 mspan 上分配内存。

- 微对象`（0，16B）` ---  先使用微型分配器，再依次尝试线程缓存、中心缓存和堆分配内存;
- 小对象` [16B，32KB]` --- 依次尝试使用线程缓存、中心缓存和堆分配内存
- 大对象 (32KB，+无穷) --- 直接在堆上分配内存

`mspan` 是 Go 语言的内存管理的基本单元，该结构体包含 `next` 和 `prev` 两个字段，他们分别指向前一个和后一个 mspan 。

```go
type mspan struct {
    next *mspan
    prev *mspan
    ...
}
```

串联之后的上述结构体会构成如下所示的双向链表，运行时会使用 `runtime.mSpanList` 存储双向链表的头节点和尾节点并在 mcache 以及 mcentral 中使用。

<img src="https://img1.kiosk007.top/static/images/blog/memory2.png" alt="memory2" style="zoom:50%;margin:auto;display:block;clear:both;" />

<br/>

- **页和内存**

每个 mspan 都管理 npages 个大小为 8KB 的页，这里的页不是操作系统中的内存页，它们是操作系统内存页的整数倍，该结构体会使用下面这些字段来管理内存页的分配和回收：

```go
type mspan struct {
        startAddr uintptr // 起始地址
        npages    uintptr // 页数
        freeindex uintptr

        allocBits  *gcBits
        gcmarkBits *gcBits
        allocCache uint64
        ...
}
```

- startAddr 和 npages  ： 确定该结构体管理的多个页所在的内存，每个页的大小都是 8KB
- freeindex：扫描页中空闲对象的初始索引
- allocBits 和 gcmarkBits：分别用于标记内存的占用和回收情况
- allocCache ： allocBits 的补码，可以用于快速查找内存中未被使用的内存

<br/>

`mspan` 会以两种不同的视角看待管理的内存，当结构体管理内存不足时，运行时会以页为单位向堆申请内存。

<img src="https://img1.kiosk007.top/static/images/blog/mspan.png" alt="mspan" style="zoom:93%;margin:auto;display:block;clear:both;" />

<br/>

当用户程序或者线程向 `mspan` 申请内存时，它会使用 `allocCache` 字段以对象为单位在管理的内存中快速查找待分配的空间：

 <img src="https://img1.kiosk007.top/static/images/blog/mspan2.png" alt="mspan2" style="zoom:93%;margin:auto;display:block;clear:both;" />

如果我们能在内存中找到空闲的内存单元则直接返回，当内存中不包含空闲的内存时，上一级的组件 mcache 会为调用 mcache.refill 更新内存管理单元以满足更多对象的分配内存的需求。



## 线程缓存 mcache

`mcache` 是 Go 语言中的线程缓存，它会与线程上的处理器（GMP - P） 一一绑定，每个线程，分配一个 mcache 用于处理微对象和小对象的分配，因为是每个线程独有的，所以不需要加锁。

`mcache` 在刚刚被初始化时是不包含 `mspan` 的，只用用户程序申请内存时才会从上一级组件获取新的 mspan 满足内存的分配

- mcache 会持有 tiny 相关字段用于微对象的内存分配。
- mcache 会持有 mspan 用于小对象的内存分配。

<br/>

其中比较重要的是**微分配器** TinyAllocator，线程缓存中还包含几个用于分配微对象的字段，下面这三个字段组成了微对象的分配器。专门管理 16 字节以下的对象。

```go
type mcache struct {
        tiny             uintptr
        tinyoffset       uintptr
        local_tinyallocs uintptr
}
```

微分配器只会用于分配非指针类型的内存。



<br/>



## 中心缓存 mcentral

`mcentral` 是内存分配器的中心缓存，与线程缓存不同的是，访问中心缓存中的内存管理单元需要使用**互斥锁**

```go
type mcentral struct {
        spanclass spanClass
        partial  [2]spanSet
        full     [2]spanSet
}
```

每个中心缓存都会管理某个跨度类的内存管理单元，它会同时持有两个 `runtime.spanSet` ，分别存储包含空闲对象和不包含空闲对象的内存管理单元。



<br/>

## 页堆 mheap

内存分配的核心组件，包含mcentral 和 heapArena，堆上的所有 mspan 都是从 mheap 结构分出来的。



**HeapArena**

Go 1.11 后采用稀疏内存管理。堆区的内存可以不连续，将堆区的内存分微一个个内存块（arena），通过 heapArena 管理。

在 mheap 中维护一个 heapArena 数组，记录所有的内存块。

<img src="https://img1.kiosk007.top/static/images/blog/mspan3.png" alt="mspan3" style="zoom:93%;margin:auto;display:block;clear:both;" />

<br/>

# 内存分配

堆上的所有对象都会通过调用 `runtime.newobject` 函数分配内存，该函数会调用 `runtime.mallocgc` 分配指定大小的内存空间，这也是用户程序向堆上申请内存空间的必经函数：

```go
func mallocgc(size uintptr, typ *_type, needzero bool) unsafe.Pointer {
        mp := acquirem()
        mp.mallocing = 1

        c := gomcache()
        var x unsafe.Pointer
        noscan := typ == nil || typ.ptrdata == 0
        if size <= maxSmallSize {
                if noscan && size < maxTinySize {
                        // 微对象分配
                } else {
                        // 小对象分配
                }
        } else {
                // 大对象分配
        }

        publicationBarrier()
        mp.mallocing = 0
        releasem(mp)

        return x
}
```

<br/>

上述代码使用 `runtime.gomcache` 获取线程缓存并判断申请内存的类型是否为指针。我们从这个代码片段可以看出 `runtime.mallocgc` 会根据对象的大小执行不同的分配逻辑，在前面的章节也曾经介绍过运行时根据对象大小将它们分成微对象、小对象和大对象，这里会根据大小选择不同的分配逻辑：

<br/>

# mallocgc 函数

runtime.mallocgc 是负责堆分配的关键函数，runtime 中的 new 和 make 函数都依赖它。其主要功能主要有 **辅助GC** 、**空间分配**、**位图标记**

## 辅助GC

如果程序申请堆内存时正处于GC标记阶段，那么，当下已分配的堆内存还没标记完，你这边又要分配新的堆内存。万一内存申请的速度超过了GC标记的速度，那就不妙了~

<br/>

## 空间分配

这里就需要根据要分配的空间大小，以及是否为noscan型空间来选择不同的分配策略了。先来看一下是如何选择策略的：
maxSmallSize是32KB，maxTinySize等于16B。也就是说：
（1）小于16字节，而且是noscan类型的内存分配请求，会使用tiny allocator；
（2）大于32KB的内存分配，包括noscan和scannable类型，都会采用大块内存分配器；
（3）剩下的，大于等于16B且小于等于32KB的noscan类型；以及小与等于32KB的scannable类型的分配请求，都会直接匹配预置的大小规格来分配。

```go
if size <= maxSmallSize {
  if noscan && size < maxTinySize {
    // 使用tiny allocator分配
  } else {
    // 使用mcache.alloc中对应的mspan分配
  }
} else {
  // 直接根据需要的页面数，分配大的mspan
}


```



<br/>

## 位图标记

即通过一个堆内存地址，如何找到对应的heapArena和mspan~



<br/>

# 栈内存

上面讨论的内存分配都是堆内存，如果把在堆中分配的对象改为在栈上分配，速度还会再快上1倍。



这是因为，由于每个线程都有独立的栈，所以分配内存时不需要加锁保护，而且栈上对象的尺寸在编译阶段就已经写入可执行文件了，执行效率更高！

性能至上的 Golang 语言就是按照这个逻辑设计的，即使你用 new 关键字分配了堆内存，但编译器如果认为在栈中分配不影响功能语义时，会自动改为在栈中分配。

当然，在栈中分配内存也有缺点，它有功能上的限制。一是， 栈内存生命周期有限，它会随着函数调用结束后自动释放，在堆中分配的内存，并不随着分配时所在函数调用的结束而释放，它的生命周期足够使用。二是，栈的容量有限，如 CentOS 7 中是 8MB 字节，如果你申请的内存超过限制会造成栈溢出错误（比如，递归函数调用很容易造成这种问题），而堆则没有容量限制。

**不过在 Go 语言中**，**是有逃逸策略的**。

在`Go`应用程序运行时，每个`goroutine`都维护着一个自己的栈区，这个栈区只能自己使用不能被其他`goroutine`使用。**栈区的初始大小是2KB**（比x86_64架构下线程的默认栈2M要小很多），在`goroutine`运行的时候栈区会按照需要增长和收缩，占用的内存最大限制的默认值在64位系统上是1GB。栈大小的初始值和上限这部分的设置都可以在`Go`的源码`runtime/stack.go`里找到：



<br/>

参考：

- [Golang 堆内存管理 -- 幼麟](https://www.bilibili.com/video/BV1gT4y1o7H1/?spm_id_from=333.999.0.0&vd_source=af544ba5c244ed2765ff78c8d2727eef)

- [Golang 内存分配](https://hardcore.feishu.cn/docs/doccnnKvHn4iFPXgk8ZpYmLArte)
