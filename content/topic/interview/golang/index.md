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

三色标记

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

在Go中，主要实现了两种锁：sync.Mutex(互斥锁) 以及 sync.RWMutex(读写锁)

<br/>

### Mutex的锁有哪几种模式



<br/>

### Mutex 锁底层如何实现

<br/>

### Mutex 是悲观锁还是乐观锁

<br/>

### 自旋锁是什么

<br/>

### RWMutex（读写锁）适用于什么场景？

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