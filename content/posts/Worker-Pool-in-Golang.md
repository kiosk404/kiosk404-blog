---
title: Worker Pool in Golang
author: kiosk
tags:
  - golang
categories:
  - programming
date: 2021-03-21 15:41:00
---
Go 是一个自动垃圾回收的编程语言，采用三色并发标记算法标记对象并回收。和其它没有自动垃圾回收的编程语言不同，使用 Go 语言创建对象的时候，我们没有回收 / 释放的心理负担，想用就用，想创建就创建。但是，如果你想使用 Go 开发一个高性能的应用程序的话，就必须考虑垃圾回收给性能带来的影响

<!--more-->

# Pool 

Pool 的出现，可以避免反复的创建一些对象，比如 TCP链接、数据库链接等等，这些对象创建都比较耗时，如果将创建好的对象放入到池子中，需要的时候取，不需要的时候归还池子，将是一个非常不错的实现方式。

通过创建一个 Worker Pool 来减少 goroutine 的使用。比如，我们实现一个 TCP 服务器，如果每一个连接都要由一个独立的 goroutine 去处理的话，在大量连接的情况下，就会创建大量的 goroutine，这个时候，我们就可以创建一个固定数量的 goroutine（Worker），由这一组 Worker 去处理连接，比如 fasthttp 中的[Worker Pool](https://github.com/valyala/fasthttp/blob/9f11af296864153ee45341d3f2fe0f5178fd6210/workerpool.go#L16)。


- 著名的TCP连接池实现：fatih 开发的 [fatih/pool](https://github.com/fatih/pool), 虽然已经归档，但是这是由于这个项目已经足够稳定。

- 数据库连接池实现：database/sql 可以参考这篇分析 [Golang 侧数据库连接池原理和参数调优](https://blog.csdn.net/qq_39384184/article/details/103954821)，同样使用方式直接参考 [这篇文章](https://www.jianshu.com/p/2d58243fae22)

**一句话总结：保存和复用临时对象，减少内存分配，降低 GC 压力。**

举例：
``` golang
type Student struct {
	Name   string
	Age    int32
	Remark [1024]byte
}

var buf, _ = json.Marshal(Student{Name: "Geektutu", Age: 25})

func unmarsh() {
	stu := &Student{}
	json.Unmarshal(buf, stu)
}
```

json 的反序列化在文本解析和网络通信过程中非常常见，当程序并发度非常高的情况下，短时间内需要创建大量的临时对象。而这些对象是都是分配在堆上的，会给 GC 造成很大压力，严重影响程序的性能。

**声明对象池**

只需要实现 New 函数即可。对象池中没有对象时，将会调用 New 函数创建。

``` golang
var studentPool = sync.Pool{
    New: func() interface{} { 
        return new(Student) 
    },
}
```

**Get && Put **
``` golang
stu := studentPool.Get().(*Student)
json.Unmarshal(buf, stu)
studentPool.Put(stu)
```

- Get() 用于从对象池中获取对象，因为返回值是 interface{}，因此需要类型转换。
- Put() 则是在对象使用完毕后，返回对象池。




# gammazero/workerpool

[gammazero/workerpool](https://pkg.go.dev/github.com/gammazero/workerpool) gammazero/workerpool 可以无限制地提交任务，提供了更便利的 Submit 和 SubmitWait 方法提交任务，还可以提供当前的 worker 数和任务数以及关闭 Pool 的功能。

下面做一些介绍

``` go
package main

import (
	"fmt"
	"github.com/gammazero/workerpool"
)

func main() {
	wp := workerpool.New(2)
	requests := []string{"alpha", "beta", "gamma", "delta", "epsilon"}

	for _, r := range requests {
		r := r
		wp.Submit(func() {
			fmt.Println("Handling request:", r)
		})
	}

	wp.StopWait()
}
```

- **使用提示**

排队的任务数没有上限，只有系统资源的限制。如果入站任务的数量太多，以至于无法排队等待处理那么解决方案就超出了workerpool的处理范围，应该通过在多个系统上分配负载来解决


## 使用介绍

## Submit

用户通过 `Submit(task func())` 方法提交一个 task 到 task 队列中。
task 函数默认没有返回值，如果想要有返回值，可以用管道将task 的返回值传到管道中。

提交的 task 会立即开启一个可用的worker或者新创建一个worker。如果没有可用的worker或者worker数已经达到最大，task会被放入到 task 等待队列中。当worker空闲时会从task 等待队列中取出task。

一个Worker长时间闲置时可以删除并释放资源。


这个函数非常简单，就是将收到的待执行任务放入到 task等待队列中。
``` go
func (p *WorkerPool) Submit(task func()) {
	if task != nil {
		p.taskQueue <- task
	}
}
```

还有一个变种, 支持同步等待结果
``` go
func (p *WorkerPool) SubmitWait(task func()) {
	if task == nil {
		return
	}
	doneChan := make(chan struct{})
	p.taskQueue <- func() {
		task()
		close(doneChan)
	}
	<-doneChan
}
```

## New

`New()` 函数创建了一个 worker goroutines pool 。 Max 指定了最大的worker数量，也就是最大并发的执行数，当没有新到来的 task 时，worker会逐渐减少至0.这里注意 taskQueue 是一个只有1个buffer的缓冲，task等待队列是 `waitingQueue()` 。dispatch 里实现派发任务的逻辑。

``` go
func New(maxWorkers int) *WorkerPool {
	// There must be at least one worker.
	if maxWorkers < 1 {
		maxWorkers = 1
	}

	pool := &WorkerPool{
		maxWorkers:  maxWorkers,
		taskQueue:   make(chan func(), 1),
		workerQueue: make(chan func()),
		stopSignal:  make(chan struct{}),
		stoppedChan: make(chan struct{}),
	}

	// Start the task dispatcher.
	go pool.dispatch()

	return pool
}
```


**dispatch 任务派发** 通过设置 idleTimer ，超过这个时间还没有task，就会杀掉一个worker。这个函数刚开始判断等待队列是否为 0，如果不为0，说明任务已经积压，需要将新的task传到等待队列中 ，`processWaitingQueue`函数内将 task 传给 waitQueue，并在workerQueue有buffer时将任务传给 worker Queue。


如果 waitQueue 不存在（长度为0），说明还不存在任务排队情况，会将task传给 workerQueue（如果workerQueue能把task塞进去的话），如果塞不进去
就创建一个新worker，要么worker刚好又不够了（达到最大worker数量），任务扔进 waitQueue。
``` go
// dispatch sends the next queued task to an available worker.
func (p *WorkerPool) dispatch() {
	defer close(p.stoppedChan)
	timeout := time.NewTimer(idleTimeout)
	var workerCount int
	var idle bool

Loop:
	for {
		if p.waitingQueue.Len() != 0 {
			if !p.processWaitingQueue() {
				break Loop
			}
			continue
		}

		select {
		case task, ok := <-p.taskQueue:
			if !ok {
				break Loop
			}
			// Got a task to do.
			select {
			case p.workerQueue <- task:
			default:
				// Create a new worker, if not at max.
				if workerCount < p.maxWorkers {
					go startWorker(task, p.workerQueue)
					workerCount++
				} else {
					// Enqueue task to be executed by next available worker.
					p.waitingQueue.PushBack(task)
					atomic.StoreInt32(&p.waiting, int32(p.waitingQueue.Len()))
				}
			}
			idle = false
		case <-timeout.C:
			// Timed out waiting for work to arrive.  Kill a ready worker if
			// pool has been idle for a whole timeout.
			if idle && workerCount > 0 {
				if p.killIdleWorker() {
					workerCount--
				}
			}
			idle = true
			timeout.Reset(idleTimeout)
		}
	}

	// If instructed to wait, then run tasks that are already queued.
	if p.wait {
		p.runQueuedTasks()
	}

	// Stop all remaining workers as they become ready.
	for workerCount > 0 {
		p.workerQueue <- nil
		workerCount--
	}

	timeout.Stop()
}
```

- `waitingQueue` 的目的是讲一个新的task放入到等待 task 队列。或者等待工人队列中有可用的工人时将 task 等待队列中取出 task 交给 worker 队列。

``` go
func (p *WorkerPool) processWaitingQueue() bool {
	select {
	case task, ok := <-p.taskQueue:
		if !ok {
			return false
		}
		p.waitingQueue.PushBack(task)
	case p.workerQueue <- p.waitingQueue.Front().(func()):
		// A worker was ready, so gave task to worker.
		p.waitingQueue.PopFront()
	}
	atomic.StoreInt32(&p.waiting, int32(p.waitingQueue.Len()))
	return true
}
```