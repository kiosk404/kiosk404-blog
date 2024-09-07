---
title: 'Go可以无限go下去吗'
author: kiosk
tags:
  - golang
categories:
  - programming
date: 2022-08-06 23:57:00
---

前一段时间看到知乎上的一个问题，[Golang 开发需要协程池吗？](https://www.zhihu.com/question/302981392) 看了知乎上大佬上的一些回答，大部分人的回答都是不需要，不过这里我也想表达一下自己的看法。

<!--more-->

<img src="https://img1.kiosk007.top/static/images/blog/golang_gorouting.png" alt="golang_gorouting" style="zoom: 40%; clear: both; display: block; margin: auto;" />

<br/>

首先先总结一下这个问题下面各个知乎网友们的答案;

- silsuer：这层的回答是肯定了协程池的作用，极限的性能压榨，复用goroutine减少重新创建也是可以接受的。所以是赞同协程池的。当让如果是个小项目就没必要搞什么协程池了。大型并发服务就需要慎重考虑了。（这个比较中肯）
- Angry Bugs：这层的回答是显然不需要协程池，因为他认为池化主要是为了解决频繁创建的开销，而goroutine已经够轻量级别。
- **千言千语**：这层的回答也是认为大部分场景下是不需要goroutine的，Go 语言自出生就身带“高并发”的标签，其本身就拥有及其优秀的并发量和吞吐量。

其他的答友基本上也都是反对协程池的概念。有提到小部分场景需要，但是又没有说的特别明白。因为Goroutine本身具备体积轻量、优质的GMP调度模型的特点，所以不需要协程池。

有关GMP调度可以参考我之前的文章：[Golang GMP](https://kiosk007.top/post/golang-gmp/)

<br/>

说到池这个概念，我们接触最多的是线程池，那么我们先了解下什么是线程池

# 什么是线程池？

贴一个维基百科的解释。[https://zh.wikipedia.org/wiki/%E7%BA%BF%E7%A8%8B%E6%B1%A0?wprov=sfti1](https://zh.wikipedia.org/wiki/%E7%BA%BF%E7%A8%8B%E6%B1%A0?wprov=sfti1)

**线程池** (英语：thread pool) ：一种线程的使用模式，线程过多会带来调度开销，进而影响缓存局部性和整体性能。而线程池维护着多个线程，等待着监督管理者分配可并发执行的任务。这避免了在处理短时间任务时创建和销毁线程的代价。

<br/>

说到这里，看起来大家对线程池的定义都主要是集中在是否可复用线程，减少线程创建、销毁的代价上。

<br/>

# 协程可以无限创建吗？

知乎网友 @silsuer 说道协程池有一个作用就是限制协程过多，那么我们来看下 goroutine是否可以无限开辟呢，如果做一个服务器，或者高并发业务场景。能否可以随意开辟goroutine并且放养不管呢？毕竟有强大的GC和优质的GMP调度算法。

看下面的代码：

```go
package main

import (
    "fmt"
    "math"
    "runtime"
)

func main() {
    //模拟用户需求业务的数量
    task_cnt := math.MaxInt64

    for i := 0; i < task_cnt; i++ {
        go func(i int) {
            fmt.Println("go func ", i, " goroutine count = ", runtime.NumGoroutine())
        }(i)
    }
}

```

结果：

![golang_gorouting_panic](https://img1.kiosk007.top/static/images/blog/golang_gorouting_panic.png)

最终被操作系统强制kill掉了，强制终结了该进程。所以我们迅速的开辟goroutine（**不受控的并发goroutine数量**）会在短时间占据操作系统的资源（CPU、内存、文件描述符等）。这些资源实际上是所有用户态程序共享的资源，所以大批的goroutine最终引发的灾难不仅仅是自身，还会影响其他程序。



# 如何控制goroutine数量

## 方法一：通过一个channel 来控制 goroutine。

```go
package main

import (
    "fmt"
    "math"
    "runtime"
)

func work(ch chan bool, i int) {
    fmt.Println("go func ", i, " goroutine count = ", runtime.NumGoroutine())
    <-ch
}

func main() {
    //模拟用户需求业务的数量
    task_cnt := math.MaxInt64
    //task_cnt := 10

    ch := make(chan bool, 3)

    for i := 0; i < task_cnt; i++ {
        ch <- true
        go work(ch, i)
    }
}

```

<br/>

运行结果

```
go func  409202  goroutine count =  4
go func  409203  goroutine count =  4
go func  409204  goroutine count =  4
go func  409205  goroutine count =  4
go func  409206  goroutine count =  4
go func  409207  goroutine count =  4
go func  409208  goroutine count =  4
go func  409209  goroutine count =  4
go func  409210  goroutine count =  4
go func  409211  goroutine count =  4
go func  409212  goroutine count =  4
go func  409213  goroutine count =  4
go func  409214  goroutine count =  4
...
```

<br/>

goroutine被成功限制到并发3个（这里打印4是因为还有一个 main goroutine）。但是这里实际上是限制了并发速度的。

程序执行的速度取决于 `work()` 函数的执行速度，这样实际上，同一时间内运行的goroutine的数量与channel限制buffer的数量一致了，从而达到限定效果。

不过这个代码有个小问题，就是如果 go_cnt 的数量如果变得小一点，就会出现打印的结果不正确的情况。

```go
package main

import (
    "fmt"
    "runtime"
)

func work(ch chan bool, i int) {

    fmt.Println("go func ", i, " goroutine count = ", runtime.NumGoroutine())
    <-ch
}

func main() {
    //模拟用户需求业务的数量
    //task_cnt := math.MaxInt64
    task_cnt := 10
    ch := make(chan bool, 3)

    for i := 0; i < task_cnt; i++ {
        ch <- true
        go work(ch, i)
    }
}

```

<br/>

打印结果：

```bash
go func  2  goroutine count =  4
go func  3  goroutine count =  4
go func  4  goroutine count =  4
go func  5  goroutine count =  4
go func  0  goroutine count =  4
go func  6  goroutine count =  4
go func  7  goroutine count =  4
```

这是因为 main 将全部的 goroutine 开辟完成之后就直接退出了，如果希望所有的main都执行需要阻塞掉main。

<br/>

## 方法二：添加 sync.WaitGroup

```go
import (
    "fmt"
    "math"
    "sync"
    "runtime"
)

var wg = sync.WaitGroup{}

func work(ch chan bool, i int) {

    fmt.Println("go func ", i, " goroutine count = ", runtime.NumGoroutine())
    <- ch
    wg.Done()
}

func main() {
    //模拟用户需求业务的数量
    task_cnt := math.MaxInt64
   
    for i := 0; i < task_cnt; i++ {
		wg.Add(1)
        ch <- true 
        go work(i)
    }
	wg.Wait()
}
```

<br/>

## 方法三：利用无缓冲 channel 与任务发送\执行分离的方式。

```go 
package main

import (
    "fmt"
    "math"
    "sync"
    "runtime"
)

var wg = sync.WaitGroup{}

func work(ch chan int) {

    for t := range ch {
        fmt.Println("go task = ", t, ", goroutine count = ", runtime.NumGoroutine())
        wg.Done()
    }
}

func sendTask(task int, ch chan int) {
    wg.Add(1)
    ch <- task
}

func main() {

    ch := make(chan int)   //无buffer channel

    goCnt := 3              //启动goroutine的数量
    for i := 0; i < goCnt; i++ {
        //启动go
        go work(ch)
    }

    taskCnt := math.MaxInt64 //模拟用户需求业务的数量
    for t := 0; t < taskCnt; t++ {
        //发送任务
        sendTask(t, ch)
    }

	  wg.Wait()
}
```

执行结果：

```
go task =  261428 , goroutine count =  4
go task =  261430 , goroutine count =  4
go task =  261434 , goroutine count =  4
go task =  261435 , goroutine count =  4
go task =  261433 , goroutine count =  4
go task =  261432 , goroutine count =  4
go task =  261438 , goroutine count =  4
go task =  261437 , goroutine count =  4
go task =  261440 , goroutine count =  4
go task =  261441 , goroutine count =  4
go task =  261442 , goroutine count =  4
```

<br/>

# go协程池

说到这里，可能大家已经知道了，协程池还是需要的，但是go 中的协程池不再是重要为了解决协程频繁创建销毁的性能问题了。

gorouting 是一种轻量级用户态线程，可以在线程的基础上实现**分时复用**，适用于IO密集型应用，goroutine 具有轻量、资源占用小、切换开销小的优势，但是受资源限制，协程创建的数量也是有限的，过多的创建协程，会导致程序消耗过多的系统资源，从而影响程序的正常运行。

<br/>

[ants](https://link.juejin.cn/?target=https%3A%2F%2Fgithub.com%2Fpanjf2000%2Fants)是一个广泛使用的goroute池，可以有效控制协程数量，防止协程过多影响程序性能。

## ants 介绍 

ants 协程池使用起来非常简单，支持默认协程池 `defaultAntsPool`、自定义协程池 `NewPool(size, options)`、指定方法协程池 `NewPoolWithFunc` 。

<br/>

```
// 使用默认的协程池
_ = ants.Submit(func() {
    fmt.Println("hello")
})

// 使用自定义协程池
p, _ := ants.NewPool(1000)
_ = p.Submit(func() {
    fmt.Println("hello")
})

// 使用指定方法的协程池
p, _ := ants.NewPoolWithFunc(1000, func(i interface{}) {
    fmt.Println(i)
})
_ = p.Invoke(Param)
```



## 实现原理

在 antsPool 协程池中，每一个worker 对应一个 gorouting。在worker的初始化阶段会通过 go 关键字创建一个 gorouting，然后这个 gorouting 会不断监听并且执行 taskChan 里面的task，类似上面的方案三。



## ants 的使用

- **Pool**

```go
import (
    "fmt"
    "sync"
    "sync/atomic"
    "time"
 
    "github.com/panjf2000/ants/v2"
)
 
func demoFunc() {
    time.Sleep(10 * time.Millisecond)
    fmt.Println("Hello World!")
}
 
func main() {
    // 释放ants的默认协程池
    defer ants.Release()
 
    var wg sync.WaitGroup
    // 任务函数
    syncCalculateSum := func() {
        demoFunc()
        wg.Done()
    }
    for i := 0; i < 100; i++ {
        wg.Add(1)
        // 提交任务到默认协程池
        _ = ants.Submit(syncCalculateSum)
    }
    wg.Wait()
    fmt.Printf("running goroutines: %d\n", ants.Running())
    fmt.Printf("finish all tasks.\n")
}
```

<br/>

- **PoolWithFunc**

```go
import (
    "fmt"
    "sync"
    "sync/atomic"
    "time"
 
    "github.com/panjf2000/ants/v2"
)
 
var sum int32
 
func myFunc(i interface{}) {
    n := i.(int32)
    atomic.AddInt32(&sum, n)
    fmt.Printf("run with %d\n", n)
}
 
func main() {
    // 初始化协程池
    p, _ := ants.NewPoolWithFunc(10, func(i interface{}) {
        myFunc(i)
        wg.Done()
    })
    // 释放协程池
    defer p.Release()
    // 提交任务
    for i := 0; i < runTimes; i++ {
        wg.Add(1)
        _ = p.Invoke(int32(i))
    }
    wg.Wait()
    fmt.Printf("running goroutines: %d\n", p.Running())
    fmt.Printf("finish all tasks, result is %d\n", sum)
}
```



## ants 协程池的配置

ants 协程池支持以下配置项：

- 协程池大小设置
- worker 协程过期时间，过期后的协程会被清理
- 内存预分配，是否初始化Pool的时候分配worker资源
- 最大阻塞任务数，默认0，任务阻塞知道分配到worker；当非0时，达到最大任务数，不执行任务并返回 overload
- 非阻塞提交，默认否，任务无worker执行时阻塞，当为true时，无worker则立即返回overload
- 自定义 panic 处理函数
- 自定义日志函数

代码片段：

```go
// 设置配置项
    options := Options{}
 options.ExpiryDuration = time.Duration(10) * time.Second
 options.Nonblocking = true
 options.PreAlloc = true
    // 初始化
 poolOpts, _ := NewPool(10, WithOptions(options))
```

