---
title: Go 并发模式
author: kiosk
draft: true
tags: 
  - devops
categories:
  - Linux
date : 2024-11-05 08:50:00
---

Go 语言最吸引人的地方时它内建的 并发支持，这篇文章不做高谈阔论，只总结部分在平时用到的常见的并发编程技巧。



<!--more-->



# 扇出模式 （Fan-out）

利用 [k8s.io/apimachinery/pkg/util/wait](http://k8s.io/apimachinery/pkg/util/wait)的 wait.Group ，将任务分给多个 worker 去完成

**多个goroutine从同一个通道读取数据，直到该通道关闭。**OUT是一种张开的模式，所以又被称为扇出，可以用来分发任务。

![golang-fan-out](https://img1.kiosk007.top/static/images/blog/20241110232432-golang-fan-out.png)

 ```go
 func FanOut(work []int, workers int) { 
 	jobs := make(chan int) 
 	ag := wait.Group{} // 启动 workers 
 	for i := 0; i < workers; i++ { 	
 		id := i 	
 		ag.Start(func() { 		
 			for job := range jobs { 			
 				fmt.Printf("Worker %d processing job %d\n", id, job) 		
 			} 	
 		}) 
 	} 	// 分发工作 
 	for _, job := range work { 	
 		jobs <- job 
 	}
 	close(jobs) 
 	ag.Wait() 
 }
 ```



<bt/>

# 扇入模式（Fan-in）

**1个goroutine从多个通道读取数据，直到这些通道关闭。**IN是一种收敛的模式，所以又被称为扇入，用来收集处理的结果。



![golang-fan-in](https://img1.kiosk007.top/static/images/blog/20241111094427-golang-fan-in.png)



```go
func FanIn(inputs ...<-chan int) <-chan int {
	output := make(chan int)
	var ag wait.Group

	// 为每个输入启动一个 goroutine
	for _, input := range inputs {
		inputCh := input
		ag.Start(func() {
			for val := range inputCh {
				output <- val
			}
		})
	}

	// 当所有输入处理完毕后关闭输出
	go func() {
		ag.Wait()
		close(output)
	}()

	return output
}
```

<br/>

# 消息消费类型

假设如下场景，生产者会生产不同类型、不同型号的数据。 

需要消费者对生产数据做整合，不同的消费者对不同的型号数据做单独的加工，

 如 同时产生 节点、域名 相关的数据，节点和域名还有向下细分的 节点1、节点2、节点3，域名也有类似的 域名1，域名2，域名3 以此类推， 需要对这些不同的数据进行单独的处理，比如累加。 如下的例子， 生产端会有 node1-3 和 domain1-3 会并发产生一些数据，消费端会以具体的一个产品为维度去消费这些数据，对数据中的数字进行累加。

<br/>

以下是数据的定义：

```go
type MsgData struct {
	Type  string `json:"type"`
	Data  string `json:"data"`
	Count int64  `json:"count"`
}

const MsgTypeDomain = "msg_domain"
const MsgTypeNode = "msg_node"
```

<br/>

首先定义 生产端的逻辑

> 其逻辑是 用3个worker，并发生成 3个 domain 和 3个node数据，其中，domain 1-3 各10条数据，node 1-3 各5条数据。

```go
func ProcessData() <-chan MsgData {
	c := make(chan MsgData, processRecordQueueSize*10)
	ag := wait.Group{}
	for worker := 0; worker < 3; worker++ {
		workerID := worker
		ag.Start(func() {
			for i := 0; i < 10; i++ {
				c <- MsgData{MsgTypeDomain, fmt.Sprintf("domain_%d", workerID), int64(i)}
			}
		})
		ag.Start(func() {
			for i := 0; i < 5; i++ {
				c <- MsgData{MsgTypeNode, fmt.Sprintf("node_%d", workerID), int64(i)}
			}
		})
	}

	go func() {
		ag.Wait()
		close(c)
	}()

	return c
}
```

<br/>

下面是数据消费端

```go
const (
	goroutineMaxIdleTimeout = 3
	processRecordQueueSize  = 50
)

var instance *Handler
var once sync.Once

// 单例模式
func Instance() *Handler {
	once.Do(func() {
		var err error
		instance = new(Handler)
		instance.stop = make(chan struct{})
		if instance.domainPool, err = ants.NewPool(100); err != nil {
			panic(err)
		}
		if instance.nodePool, err = ants.NewPool(100); err != nil {
			panic(err)
		}
	})
	return instance
}

type Handler struct {
	stop             chan struct{}
	nodeProcessors   sync.Map
	domainProcessors sync.Map
	nodePool         *ants.Pool
	domainPool       *ants.Pool
}

func (instance *Handler) ConsumerData(data MsgData) {
	switch data.Type {
	case MsgTypeDomain:
		instance.handleDomainData(data)
	case MsgTypeNode:
		instance.handleNodeData(data)
	default:
		log.Fatal("unknown msg type")
	}
}

func (instance *Handler) handleNodeData(data MsgData) {
	newNode := &recordStat{recordName: data.Data, ch: make(chan MsgData, processRecordQueueSize)}
	if v, ok := instance.nodeProcessors.LoadOrStore(data.Data, newNode); ok {
		if record, ok := v.(*recordStat); ok {
			go record.processResult(data)
		}
	} else {
		if err := instance.nodePool.Submit(newNode.run); err != nil {
			log.Fatalf("node produce goroutine task submit failed, %s", err.Error())
			return
		}
		newNode.processResult(data)
	}
}

func (instance *Handler) handleDomainData(data MsgData) {
	newDomain := &recordStat{recordName: data.Data, ch: make(chan MsgData, processRecordQueueSize)}
	if v, ok := instance.nodeProcessors.LoadOrStore(data.Data, newDomain); ok {
		if record, ok := v.(*recordStat); ok {
			go record.processResult(data)
		}
	} else {
		if err := instance.nodePool.Submit(newDomain.run); err != nil {
			log.Fatalf("domain produce goroutine task submit failed, %s", err.Error())
			return
		}
		newDomain.processResult(data)
	}
}
```

这里的消费端会从 sync.Map 中加载出一个具体负责实现对具体 产品的处理逻辑的 结构体，使用 `LoadOrStore` 结构防止并发访问时出现问题。

<br/>

```go
type recordStat struct {
	recordName    string
	count         int64
	ch            chan MsgData
	lastReceiveAt time.Time
}

func (record *recordStat) do(data MsgData) {
	atomic.AddInt64(&record.count, data.Count)
}

func (record *recordStat) processResult(data MsgData) {
	record.ch <- data
}

func (record *recordStat) destroy() {
	Instance().nodeProcessors.Delete(string(record.recordName))
	Instance().domainProcessors.Delete(string(record.recordName))
	log.Printf("record(%s) destoryed finnal count %d \n", record.recordName, record.count)

	// 没有在运行的任务了，退出
	isRemaining := true
	Instance().nodeProcessors.Range(func(key, value any) bool {
		isRemaining = false
		return false
	})
	Instance().domainProcessors.Range(func(key, value any) bool {
		isRemaining = false
		return false
	})
	if isRemaining {
		Instance().stop <- struct{}{}
	}
}

func (record *recordStat) run() {
	timer := time.NewTimer(goroutineMaxIdleTimeout * time.Second)
	defer timer.Stop()
	defer record.destroy()
	for {
		select {
		case data, ok := <-record.ch:
			if ok {
				record.lastReceiveAt = time.Now()
				record.do(data)
				if !timer.Stop() {
					<-timer.C
				}
				timer.Reset(goroutineMaxIdleTimeout * time.Second)
			} else {
				log.Printf("(%s) goroutine is destroyed because data channel close, last receive time %s \n",
					record.recordName, record.lastReceiveAt.Format(time.Stamp))
				return
			}
		case <-timer.C:
			//log.Printf("(%s) goroutine is destroyed because no data was received for a long time, last receive time %s \n", record.recordName, record.lastReceiveAt.String())
			return
		}
	}
}
```



