---
title: Golang
subtitle: Build fast, reliable, and efficient software at scale
disallow: true
lightgallery: true
comment: false
featuredImage: https://img1.kiosk007.top/static/images/go/golang.png
date: 2022-06-08 23:41:26
---

> Based on google



- [sample](/topic/interview/golang/#sample)
  - [new 和 make 的区别？](topic/interview/golang/#new-和-make-的区别)
  - [Golang 的参数值传递还是引用传递?](topic/interview/golang/#golang-的参数值传递还是引用传递)
- [medium](/topic/interview/golang/#medium)
  - [Golang 的内存管理](/topic/interview/golang/#golang-的内存管理)
    - [Go 是如何分配内存的](/topic/interview/golang/#go-是如何分配内存的)
    - [Golang 的GC过程](/topic/interview/golang/#golang-的gc-过程)
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
    - [读一个已关闭的channel会怎样、没初始化的channel写会怎样](/topic/interview/golang/#读一个已关闭的channel会怎样没初始化的channel写会怎样)
    - [已关闭的channel写数据会怎样，如何判断一个channel已关闭](/topic/interview/golang/#已关闭的channel写数据会怎样如何判断一个channel已关闭)
    - [select case 中有2个case 读channel，其中一个关闭，读数据会怎样](/topic/interview/golang/#select-case-中有2个case-读channel其中一个关闭读数据会怎样)

## sample

### new 和 make 的区别？

Go语言中 new 和 make 是两个内置函数，主要用来创建并分配类型的内存。

new 只分配内存，而 make 只能用于 slice、map 和 channel 的初始化。

new返回的是他们的指针，而make 返回的是是他们类型的本身，因为chan、make、slice 本身就是引用类型。

<br/>

### Golang 的参数值传递还是引用传递

golang 默认使用的是值传递，即拷贝传递，也就是深拷贝。只有一些特定的类型，如 slice、map、channel、function、pointer 这种天生就是指针的类型是通过引用传递的。

> 但是传指针会发生逃逸，会导致本来应该分配到栈上的内存逃逸到堆上。

<br/>

## medium

### Golang 的内存管理

#### Go 是如何分配内存的

内存空间包含两个重要的区域：栈（stack）和堆（heap），Go 语言的内存分配由标准库自动完成。

**小内存分配**：当变量需要小于 32 KiB 内存时，Go 语言会从名为 `mcache` 的本地缓存中为小于 32 KiB 的需求申请内存，该缓存处理内存块大小为 32 KiB 的可分配内存的链表，名为 `mspan`

**大内存分配**：Go直接分配大内存到堆上

详见：[https://zhuanlan.zhihu.com/p/352133292](https://zhuanlan.zhihu.com/p/352133292)

<br/>

#### Golang 的GC 过程

非常古老的Go版本采用“标记清除法”，即通过标记清除法进行垃圾回收。

后来的 Golang 采用三色标记法进行垃圾回收。

1. 初始状态下都是白色对象
2. 从根节点开始遍历，把遍历到的对象变成灰色对象
3. 遍历灰色对象，将灰色对象引用的对象也标记为灰色，再将遍历过的灰色对象变成黑色。
4. 重复步骤3
5. 通过 “混合写屏障“ 检测对象有变化，重复以上操作
6. 清除所有的白色对象



混合写屏障机制，栈空间每次GC时直接全部标为黑色，堆空间启用混合写屏障，没有 STW。



详见：[https://kiosk007.top/post/golang-gc/](https://kiosk007.top/post/golang-gc/)

<br/>



#### 什么是强弱三色不变式

强三色不变式：强制性不允许黑色对象引用白色

弱三色不变式：黑色对象可以引用白色对象，白色对象需要存在其他灰色的引用。

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

### 协程阻塞，调度器会怎么做？

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

### 若干个 goroute ，其中一个 panic ，会发生什么，defer 可以捕获子 goroute 的panic 吗？

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

### 读一个已关闭的channel会怎样、没初始化的channel写会怎样？

读到空值，崩溃

<br/>

### 已关闭的channel写数据会怎样？如何判断一个channel已关闭？

崩溃，if _, ok := <- ch  的形式

<br/>

### select case 中有2个case 读channel，其中一个关闭，读数据会怎样。

每次 select 都是随机读的，即便有已经关闭的channel，依旧还是会读到。

<br/>