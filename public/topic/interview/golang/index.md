# Golang


> Based on google



- [sample](/topic/interview/golang/#sample)
  - [new 和 make 的区别？](/topic/interview/golang/#new-和-make-的区别)
  - [Golang 的参数值传递还是引用传递?](/topic/interview/golang/#golang-的参数值传递还是引用传递)
  - [Golang Slice](/topic/interview/golang/#golang-slice)
  - [interface{}变量和nil做比较](/topic/interview/golang/#interface{}变量和nil做比较)
  - [go中 import/const/var/init/main 的执行顺序](/topic/interview/golang/#go中-importconstvarinitmain-的执行顺序)
  - [defer的执行顺序](/topic/interview/golang/#defer的执行顺序)
  - [什么是 go 的内存逃逸](/topic/interview/golang/#什么是go的内存逃逸)
- [medium](/topic/interview/golang/#medium)
  - [Golang 的内存管理](/topic/interview/golang/#golang-的内存管理)
    - [Go 是如何分配内存的](/topic/interview/golang/#go-是如何分配内存的)
    - [Golang 的GC过程](/topic/interview/golang/#golang-的-gc-过程)
    - [什么是强弱三色不变式](/topic/interview/golang/#什么是强弱三色不变式)
    - [GC的时机是什么时候](/topic/interview/golang/#gc的时机是什么时候)
  - [GMP](topic/interview/golang/#gmp)
    - [协程阻塞，调度器会怎么做？](topic/interview/golang/#协程阻塞调度器会怎么做)
    - [M的流转状态](/topic/interview/golang/#m的流转状态)
    - [一个goroute会占用多少内存，假设多个goroute一直占用资源会怎样](/topic/interview/golang/#一个goroute会占用多少内存假设多个goroute一直占用资源会怎样)
    - [一个 goroute 发生OOM 会怎样？](/topic/interview/golang/#一个-goroute-发生oom-会怎样)
    - [若干个 goroute，其中一个 panic，会发生什么，defer可以捕获子goroute的panic吗？](/topic/interview/golang/#若干个-goroute-其中一个-panic-会发生什么defer-可以捕获子-goroute-的panic-吗)
  - [反射](/topic/interview/golang/#反射)
    - [反射如何获取字段tag](/topic/interview/golang/#反射如何获取字段-tag)
    - [如何通过反射修改值](/topic/interview/golang/#如何通过反射修改值)
  - [锁](/topic/interview/golang/#锁)
    - [golang 的锁机制](/topic/interview/golang/#golang-的锁机制)
    - [Mutex的锁有哪几种模式](/topic/interview/golang/#mutex的锁有哪几种模式)
    - [Mutex锁底层如何实现](/topic/interview/golang/#mutex-锁底层如何实现)
    - [Mutex是悲观锁还是乐观锁](/topic/interview/golang/#mutex-是悲观锁还是乐观锁)
    - [RWMutex(读写锁)适用于什么场景](/topic/interview/golang/#rwmutex读写锁适用于什么场景)
  - [channel](/topic/interview/golang/#channel)
    - [channel 的内部实现是怎么样的?](/topic/interview/golang/#channel-的内部实现是怎么样的)
    - [读一个已关闭的channel会怎样、没初始化的channel写会怎样](/topic/interview/golang/#读一个已关闭的channel会怎样没初始化的channel写会怎样)
    - [已关闭的channel写数据会怎样，如何判断一个channel已关闭](/topic/interview/golang/#已关闭的channel写数据会怎样如何判断一个channel已关闭)
    - [select case 中有2个case 读channel，其中一个关闭，读数据会怎样](/topic/interview/golang/#select-case-中有2个case-读channel其中一个关闭读数据会怎样)
    - [golang如何限制并发](/topic/interview/golang/#golang如何限制并发)
  - [context](/topic/interview/golang/#context)
    - [golang中的context有什么用](/topic/interview/golang/#golang中的context有什么用)
    - [context.Background的意义](/topic/interview/golang/#contextbackground-的意义)
    - [WithCancel() 和 WithTimeout() 可以通知多个goroutine, 如何实现的](/topic/interview/golang/#withcancel-和-withtimeout-可以通知多个goroutine-如何实现的)
  - [map](/topic/interview/golang/#map)
    - [map 为什么是不安全的？](/topic/interview/golang/#map-为什么是不安全的)

## sample

### new 和 make 的区别？

Go语言中 new 和 make 是两个内置函数，主要用来创建并分配类型的内存。

new 只分配内存，而 make 只能用于 slice、map 和 channel 的初始化。

new返回的是他们的指针，指针指向分配类型的内存地址。而make 返回的是他们类型的本身，因为chan、map、slice 本身就是引用类型，所以没有必要再返回他们的指针。

<br/>

### Golang 的参数值传递还是引用传递

golang 默认使用的是值传递，即拷贝传递，也就是深拷贝。只有一些特定的类型，如 slice、map、channel、function、pointer 这种天生就是指针的类型是通过引用传递的。

**但是传指针会发生逃逸，会导致本来应该分配到栈上的内存逃逸到堆上。**

<br/>

### Golang Slice 
1. 底层是如何实现的？
>答：切片本身并不是动态数组或者数组指针。它内部实现的数据结构通过指针引用底层数组
>
>```go
>type slice struct {
>	array unsafe.Pointer
>	len   int
>	cap   int
>}
>```
>切片的结构体由3部分构成，Pointer 是指向一个数组的指针，len 代表当前切片的长度，cap 是当前切片的容量。cap 总是大于等于 len 的。
>
> 详见：[https://halfrost.com/go_slice/](https://halfrost.com/go_slice/)


2. 如何扩容

> 当前所需容量大于原先容量的2倍时，则申请当前所需的容量。
> 
> 如果上述条件不满足，则进行如下判断
> 
> 原切片长度小于1024则申请原先容量的2倍
> 
> 否则每次增加 1/4 ，直到大于所需的容量为止。
>
> 详见：[https://halfrost.com/go_slice/](https://halfrost.com/go_slice/)

3. 判断 cap 的技巧
> 核心问题：如果append需要扩容，就会完全开辟一个新空间。
>
> 详见：[https://coolshell.cn/articles/21128.html](https://coolshell.cn/articles/21128.html)

<br/>

### interface{}变量和nil做比较

一个 []string 切片的空值经过函数调用后再比较会不等于 nil, 

Go 语言中有两种略微不同的接口，一种是带有一组方法的接口，另一种是不带任何方法的空接口 interface{}。

Go 语言使用`runtime.iface`表示带方法的接口，使用`runtime.eface`表示不带任何方法的空接口interface{}。如下是 runtime.eface 的定义。
``` go
type eface struct { // 16 字节
	_type *_type
	data  unsafe.Pointer
}
```
而从空接口的定义得知需要两个指针均为0才行，但是经过一次传递会导致其变成一个值为空结构体 slice 的一个变量。

空切片为 nil 因为其值为零值，类型为 []string 的空切片传给空接口后，因为空接口的值并不是零值，所以接口变量不是 nil。

详见：[https://cloud.tencent.com/developer/article/1911289](https://cloud.tencent.com/developer/article/1911289)
 

<br/>

### go中 import/const/var/init/main 的执行顺序
import > const > var > init > main

<br/>

### defer的执行顺序
多个defer出现的时候，它是一个栈的关系，先进后出

当return 和 defer 同时出现时，**先调用return再调用defer**

详见: [了解defer的执行顺序](https://kiosk007.top/post/%E4%BA%86%E8%A7%A3golang%E7%9A%84defer/)

<br/>

### 什么是go的内存逃逸
在传统的编程语言里，会根据程序员指定的方式来决定变量内存分配是在栈还是堆上，比如声明的变量是值类型，则会分配到栈上，或者 new 一个对象则会分配到堆上。

在 Go 里变量的内存分配方式则是由编译器来决定的。如果变量在作用域（比如函数范围）之外，还会被引用的话，那么称之为发生了逃逸行为，此时将会把对象放到堆上，即使声明为值类型；如果没有发生逃逸行为的话，则会被分配到栈上，即使 new 了一个对象。

<br/>

## medium

### Golang 的内存管理

#### Go 是如何分配内存的

Go 的内存分配借鉴了 Google 的 TCMalloc 分配算法，其核心思想是 **内存池 + 多级对象管理**。内存池主要是预先分配内存，减少向系统申请的频率；多级对象有：mheap、mspan、arenas、mcentral、mcache。
<br/>

**大内存的分配**: 当要分配大于 32K 的对象时，从 mheap 分配。
**中等内存的分配**: 当要分配的对象小于等于 32K 大于 16B 时，从 P 上的 mcache 分配，如果 mcache 没有内存，则从 mcentral 获取，如果 mcentral 也没有，则向 mheap 申请，如果 mheap 也没有，则从操作系统申请内存。
**小内存的分配**:当要分配的对象小于等于 16B 时，从 mcache 上的微型分配器上分配。

<br/>

类比下来，所有的内存分配相关库包括操作系统本身都是类似的逻辑。
比如 Linux 操作系统的内存分配有2个非常有名的算法 **Buddy System** 和 **Slab 调度器** 。
<br/>

**Buddy System**：
>Buddy System 是 Linux 内核中用于内存管理的一种算法，主要用于管理物理内存。它将内存分割成大小为 2 的幂次方的块，例如 1KB, 2KB, 4KB 等。当应用程序请求内存时，Buddy System 会找到第一个大于或等于请求大小的内存块，并将其分割成更小的块来满足请求。这种分割过程是递归的，直到找到合适的内存块。当内存被释放时，Buddy System 会尝试将相邻的空闲内存块合并，以减少内存碎片。
<br/>

**Slab Allocator**：
>Slab Allocator 是 Linux 内核中用于管理内核内存的机制。它主要用于分配和释放内核对象，如进程描述符、文件对象等。Slab Allocator 将内存分为多个缓存（slab cache），每个缓存用于特定类型的内核对象。当内核需要分配一个对象时，它会从相应的缓存中获取内存。当对象不再需要时，内存会被返回到缓存中，而不是直接释放回物理内存。这样可以减少内存碎片，提高内存分配和释放的效率。
<br/>

**Buddy System 和 Slab Allocator 的关系**：
>Buddy System 和 Slab Allocator 都是 Linux 内核中的内存管理机制，但它们的作用域和目标不同。Buddy System 主要用于管理物理内存，而 Slab Allocator 专注于内核内存的管理。在某些情况下，Slab Allocator 可能会使用 Buddy System 来分配和释放物理内存。例如，当 Slab Allocator 需要更多的内存来扩展缓存时，它可能会通过 Buddy System 来请求物理内存。同样，当 Slab Allocator 释放内存时，它可能会将内存返回给 Buddy System，以便重新分配给其他用途。
<br/>

总的来说，Buddy System 和 Slab Allocator 都是 Linux 内核中重要的内存管理机制，它们相互协作，共同确保内存的有效利用和管理。

<br/>

**TCMalloc**（Thread-Caching Malloc）是一个由 Google 开发的内存分配器，旨在提高多线程程序中的内存分配性能。它的工作原理和与 Buddy System 和 Slab Allocator 的关联如下：
<br/>

**TCMalloc 的工作原理**：
>线程缓存：TCMalloc 为每个线程提供一个小的缓存区域，用于存储该线程分配的内存块。这样可以减少锁的争用，因为每个线程可以独立地从自己的缓存中分配和释放内存。
>中央缓存：当线程缓存中的内存块用尽时，TCMalloc 会从中央缓存中请求更多的内存块。中央缓存是一个全局的内存池，用于存储不同大小的内存块。
>页面分配：当中央缓存中的内存也不足时，TCMalloc 会直接从操作系统请求更多的内存页面。这通常涉及到与操作系统的内存管理机制（如 Buddy System）交互。
<br/>

**与 Buddy System 的关联**：
>TCMalloc 在请求大量内存时，可能会与操作系统的物理内存管理机制（Buddy System）交互。操作系统通过 Buddy System 分配物理内存页面给 TCMalloc，然后 TCMalloc 将这些页面划分为更小的内存块，并将其存储在中央缓存中。
<br/>

**与 Slab Allocator 的关联**：
>TCMalloc 主要用于用户空间的内存分配，而 Slab Allocator 是内核空间的内存分配机制。尽管两者在内存分配上有不同的应用场景，但它们都可以从操作系统的物理内存管理机制中受益。
>在某些情况下，如果 TCMalloc 需要为内核对象分配内存，它可能会间接地与 Slab Allocator 交互。例如，当 TCMalloc 需要扩展其中央缓存时，它可能会请求内核分配更多的物理内存，而内核可能会使用 Slab Allocator 来满足这一请求。

总的来说，TCMalloc 是一个高效的用户空间内存分配器，它通过减少锁的使用和优化内存块的分配和释放来提高性能。虽然 TCMalloc 主要与用户空间的内存分配相关，但它在需要时也会与操作系统的物理内存管理机制（如 Buddy System）以及内核空间的内存分配机制（如 Slab Allocator）交互。


详见：[内存分配与回收](https://kiosk007.top/post/%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F%E7%9F%A5%E8%AF%86%E6%B1%87%E6%80%BB/#%E5%86%85%E5%AD%98%E5%88%86%E9%85%8D%E4%B8%8E%E5%9B%9E%E6%94%B6)

<br/>

#### Golang 的 GC 过程

非常古老的Go版本采用“标记清除法”，即通过标记清除法进行垃圾回收。

后来的 Golang 采用三色标记法进行垃圾回收。

1. 初始状态下都是白色对象
2. 从根节点开始遍历，把遍历到的对象变成灰色对象
3. 遍历灰色对象，将灰色对象引用的对象也标记为灰色，再将遍历过的灰色对象变成黑色。
4. 重复步骤3
5. 通过 “混合写屏障“ 检测对象有变化，重复以上操作
6. 清除所有的白色对象

<br/>

混合写屏障机制，栈空间每次GC时直接全部标为黑色，堆空间启用混合写屏障，没有 STW。

**写屏障机制**：避免被没有被引用的对象，又被黑色对象引用而导致的误删除。这引入了 强三色不变式和弱三色不变式。而**插入写屏障**是以强三色不变式为思路。**删除写屏障** 是以弱三色不变式为思路。

插入写屏障：A对B进行引用。B直接被标记为灰色。有个缺点。栈上的对象没法捕捉，没有生成写屏障。所以导致栈上还要STW扫描一遍。

删除写屏障：被删除的对象自身是灰色或者白色，都会被标记为灰色。缺点是回收精度低，一个对象被删除了，但他引用的对象依旧可以存活过一轮GC。

Go v1.8 混合写屏障。栈上不启用GC。



详见：[https://kiosk007.top/post/golang-gc/](https://kiosk007.top/post/golang-gc/)

<br/>



#### 什么是强弱三色不变式

强三色不变式：强制性不允许黑色对象引用白色

弱三色不变式：黑色对象可以引用白色对象，但白色对象需要存在其他灰色的引用。

<br/>

#### GC的时机是什么时候

GC 触发的场景主要分两大类，分别是：

1. 系统触发：运行时自行根据内置的条件检查触发
2. 手动触发：代码中调用 runtime.GC 的方法来触发 GC 行为

其中系统触发场景中。1. 当所分配的堆大小达到阈值时会触发; 2. 当距离上一个 GC 周期的时间超过一定时间会触发; 



<br/>

### GMP

G 是 gorouting、M 是系统级线程、P是调度器。

程序刚启动时，P会绑定M0，G被放入P的本地队列中执行，当P的本地队列满了，G可能会被放到一个全局队列中。

1. 线程M想要运行G，就必须先和一个P进行关联
2. P 获取本地队列的G，如果获取不到就从全局队列或者其他的 MP 中偷取。
3. G 运行结束后，M会从P中获取下一个 G，如此往复。

大致流程如上，还有一些细节，比如 M 与 P 是如何绑定的？G 阻塞了怎么办？M与P的关系是什么？可以具体再说。



详见：[https://kiosk007.top/post/golang-gmp/](https://kiosk007.top/post/golang-gmp/)



调度器的设计策略（基本就是为了复用）：

1. work stealing 机制：当本线程无可以运行的G时，回去其他线程绑定的P中偷取G。而不是直接销毁线程。
2. hand off 机制：当本线程上的G运行系统调用阻塞时，线程会释放绑定的P，把 P 转移给其他的空闲的线程。而阻塞的协程会留在线程之上。

<br/>

### 协程阻塞，调度器会怎么做？

hand off 机制：

当一个协程发生阻塞时，当前协程上的 M 会和 P 立即解绑，如果P上还有其他的协程G，P会唤醒一个M和他绑定，否则P会加入到空闲P列表，等待M来获取可用的P。

如果不是阻塞的系统的调用，M 和 P 还是会解绑，只不过 M 会记住 P。当 G 和 M 退出系统调用时，会获取之前的P，获取不到的话，当前的 G 加入全局队列，M加入可休眠列表。

<br/>

### M的流转状态

M 线程会有两种状态，自旋 和 非自旋。

<br/>

### 一个goroute会占用多少内存，假设多个goroute一直占用资源会怎样？

一个 gorouting 占用8K，老版本中一个 gorouting 一直占用资源会导致 P （因为P最大为 GOMAXPROC 个）耗尽，最终导致程序卡死。因为之前是非抢占式，Go 1.14 之后变成了抢占式，基于系统信号的抢占。

<br/>

### 一个 goroute 发生OOM 会怎样？

没有recover的话，父线程会 panic。

<br/>

### 若干个 goroute ，其中一个 panic ，会发生什么，defer 可以捕获子 goroute 的 panic 吗？

全部崩溃，不能！

<br/>

### 反射

反射在go里调用 reflect 包，可实现通过调用获取借口值到反射对象。

### 反射如何获取字段 tag？

通过 ` t.Field(i).Tag.Get("json")` 调用

<br/>

### 如何通过反射修改值

需要对变量的指针进行值反射（reflect.ValueOf），再获取其元素值（Elem），最后进行 SetString\SetFloat 等操作

<br/>

## 锁

### golang 的锁机制

在Go中，主要实现了两种锁：sync.Mutex(互斥锁) 以及 sync.RWMutex(读写锁)。

sync.Mutex 的常用方法有2个：Mutex.Lock() 用来获取锁  、 Mutex.Unlock() 用来释放锁。

https://juejin.cn/post/7032457568433733639

<br/>

### Mutex的锁有哪几种模式

互斥锁在设计上主要有两种模式：正常模式和饥饿模式。

之所以引入了饥饿模式，是为了保证 goroutine 获取互斥锁的公平性。所谓公平性，其实就是多个 goroutine 在获取锁时，goroutine 获取锁的顺序，和请求锁的顺序一致，保持公平。

**正常模式**：所有阻塞在等待队列中的 goroutine 会按顺序进行锁的获取，当唤醒一个等待队列中的 goroutine 时，此goroutine 会和其他新的请求锁的 goroutine 去竞争，通常新请求锁的 goroutine 更容易获取锁，这是因为新请求锁的 goroutine 正在占用 CPU 片执行，大概率可以直接执行到获取锁的逻辑。

**饥饿模式**：新请求锁的goroutine不会进行锁的获取，而是加入到队列尾部阻塞等待获取锁。

**饥饿模式的出发条件**：

- 当一个goroutine等待锁的时间超过1ms时，互斥锁会切换到饥饿模式

**饥饿模式的取消条件**：

- 当获取到锁的这个goroutine是等待锁队列中的最后一个goroutine，互斥锁会切换到正常模式。
- 当获取到锁的这个goroutine的等待时间在 1ms 以内，互斥锁会切回到正常模式

<br/>

### Mutex 锁底层如何实现

Go中的sync.Mutex的结构体为：

```go
type Mutex struct {
	state int32
	sema  uint32
}
```

**sync.Mutex由两个字段构成**，**state用来表示当前互斥锁处于的状态**，**sema用于控制锁的信号量**。

互斥锁的 state 主要记录了如下四种状态：

<img src="https://img1.kiosk007.top/static/images/blog/golang_mutex.png" alt="golang_mutex" style="zoom:50%; margin: auto; clear: both; display: block;" />

- waiter_num：记录了当前等待抢这个锁的 goroutine 数量
- starving：当前锁是否处于饥饿状态
- woken：当前锁是否有goroutine已被唤醒
- locked：当前锁是否被 goroutine 持有



sema 信号量的作用：

当持有锁的goroutine释放锁后，会释放 sema 信号量，这个信号量会唤醒之前抢锁阻塞的goroutine来获取锁。



<br/>

### Mutex 是悲观锁还是乐观锁
golang 的 Mutex锁是一个悲观锁。每次去操作数据之前都会上锁，操作结束后解锁。


乐观锁在GO中的实现具体应该是 CAS。比较并交换(compare and swap, CAS)，是原子操作的一种，可用于在多线程编程中实现不被打断的数据交换操作。

```go
// CompareAndSwapUint32 executes the compare-and-swap operation for a uint32 value.
func CompareAndSwapUint32(addr *uint32, old, new uint32) (swapped bool)

```

我们有两个goroutineA和goroutineB，接下来我们简称 A 和 B， 共享资源称为C

1. A 和 B 均保存 C 当前的值
2. A 尝试使用CAS（56，53）更新C的值
3. C目前为56，可以更新，然后更新成功
4. B尝试使用CAS（56，53）更新C的值
5. C已经为53，更新失败。

<br/>

### 自旋锁是什么

某个协程持有锁的时间长，等待的协程一直循环等待，消耗CPU资源。另外其还有不公平的特点，可能存在有的协程等待时出现饥饿。

详见：[https://zhuanlan.zhihu.com/p/366437266](https://zhuanlan.zhihu.com/p/366437266)

<br/>

### RWMutex（读写锁）适用于什么场景？

`RWMutex`是一个支持**并行读串行写**的读写锁。`RWMutex`具有**写操作优先**的特点，写操作发生时，仅允许正在执行的读操作执行，后续的读操作都会被阻塞。

适用场景在 大量的并发读，少量的并发写的场景；如微服务配置更新、交易路由缓存等场景。相较于 `Mutex` 互斥锁、`RWMutex`有更好的读性能。

详见：[https://juejin.cn/post/6968853664718913543](https://juejin.cn/post/6968853664718913543)

<br/>



## channel 

### channel 的内部实现是怎么样的？

Go 语言的 channel 实现是通过 runtime.hchan 结构体，利用互斥锁、环形缓冲区和等待队列来同步 goroutines 间的通信，并保证线程安全和数据的先进先出。

[深入 Go 并发原语 — Channel 底层实现](https://halfrost.com/go_channel/)

<br/>

### 读一个已关闭的channel会怎样、没初始化的channel写会怎样？

已关闭的 channel 读到空值，没初始化的 channel 会崩溃

<br/>

### 已关闭的channel写数据会怎样？如何判断一个channel已关闭？

向已关闭的channel写数据会崩溃，可以通过 `if _, ok := <- ch`  的方式判断channel没有关闭

<br/>

### select case 中有2个case 读channel，其中一个关闭，读数据会怎样。

每次 select 都是随机读的，即便有已经关闭的channel，依旧还是会读到。

<br/>

### golang如何限制并发 
channel的方式。
详见: [https://cloud.tencent.com/developer/article/1986989](https://cloud.tencent.com/developer/article/1986989)

<br/>

## context
详见: [Golang Context](https://zhuanlan.zhihu.com/p/482953942)

### golang中的context有什么用
1. context可用于指定超时时长取消多个goroutine (使用WithTimeout方法)
2. 可以指定触发条件，取消多个 goroutine 的运行 (使用WithCancel方法)
3. 设置 key \ value（使用WithValue，Value方法）

<br/>

### context.Background() 的意义
用于new(emptyCtx),作为初始节点, emptyCtx属于int类型, 但实现了context.Context接口的所有方法, 故初始化了Context, 方便以后初始化cancelCtx, timerCtx. 

<br/>

### WithCancel() 和 WithTimeout() 可以通知多个goroutine, 如何实现的

close 单项接收管道,返回空结构体, 使select监听的ctx.Done()返回空结构体, 取消阻塞, 结束多个协程, 返回自定义的结果。

<br/>

## map

### map 为什么是不安全的？
map 在扩缩容时，需要进行数据迁移，迁移的过程并没有采用锁机制防止并发操作，而是会对某个标识位标记为 1，表示此时正在迁移数据。如果有其他 goroutine 对 map 也进行写操作，当它检测到标识位为 1 时，将会直接 panic。

如果并发读 Map 是不会有panic的，或者 Map的Value是指针且修改是并发安全的（如 value 是 &atomic.Value{} ），那么也不会 panic 。
如下：
``` go
func main() {
	a := make(map[string]*atomic.Value)
	a["a"] = &atomic.Value{}
	go func() {
		for i := 0; i < 100; i++ {
			a["a"].Store(i)
		}
	}()
	go func() {
		for i := 0; i < 100; i++ {
			fmt.Println(a["a"])
		}
	}()

	time.Sleep(2 * time.Second)
}
```
但是要是 map key 对应 value 要并发变化的则会panic.

如果我们想要并发安全的 map，则需要使用 sync.map。
[gomap怎么并发访问](https://kiosk007.top/post/gomap%E6%80%8E%E4%B9%88%E5%B9%B6%E5%8F%91%E8%AE%BF%E9%97%AE/)
