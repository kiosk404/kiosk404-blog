---
title: 实现一个Web服务-(二)
author: kiosk
draft: false
tags:
  - web
categories:
  - programming
date: 2022-04-11 10:15:00
---

[实现一个Web服务-（一）](https://kiosk007.top/post/实现一个web服务-一/) 从 net/http 基础网络库中实现了一个 Web Server，基本实现了路由匹配、上下文、中间件等特性。除这些基本的功能外，还有一些更高阶的功能。比如优雅关闭、日志等功能。



<!--more-->

# 优雅关闭（Graceful Close）

对于一个服务而言，功能和需求的开发一定是不断迭代的，在迭代的过程中，服务不可避免的会存在关闭、启动这样的动作。假设当前的服务是一个非常繁忙的服务，一个不优雅的关闭，请求的进程中断，导致用户的请求出现异常，这个在普通的博客网站没什么影响，但是如果是和支付相关的业务就存在很严重的损失。



所以，优雅关闭服务的本质近视关闭进程时不能暴力关闭进程，而是等该进程的所有请求逻辑处理完成之后再关闭。那么问题的本质就是 **“控制关闭进程的操作”**和**“如何等多所有的逻辑结束”**



**如何控制接管关闭的操作**

**信号**

在终端，非守护进程可以使用 Ctrl+C 的方式结束一个程序，其本质是发送了一个 **SIGINT** 信号， Ctrl+\ 向进程发送 **SIGQUIT** 信号，也是可以被阻塞处理，唯一的不同的是，默认行为会产生 core 文件。kill -9 pid 会给对应 pid 的进程发送 **SIGKILL** 信号，kill pid 会向进程发送 **SIGTERM** 信号。

| **信号** | **操作**  | **是否可以被处理** |
| -------- | --------- | ------------------ |
| SIGINT   | Ctrl+C    | 可以               |
| SIGQUIT  | Ctrl + \\ | 可以               |
| SIGTERM  | kill      | 可以               |
| SIGKILL  | kill -9   | 不可以             |

**os/signal 库**

go 提供了捕获信号的方法，提供如下方法。

``` go

// 忽略某个信号
func Ignore(sig ...os.Signal){}
// 判断某个信号是否被忽略了
func Ignored(sig os.Signal) bool{}
// 关注某个/某些/全部 信号
func Notify(c chan<- os.Signal, sig ...os.Signal){}
// 取消使用 notify 对信号产生的效果
func Reset(sig ...os.Signal){}
// 停止所有向 channel 发送的效果
func Stop(c chan<- os.Signal){}
```



因为使用 Ctrl 或者 kill 命令，它们发送的信号是进入 main 函数的，**所以即只有 main 函数所在的 Goroutine 监听信号。**



## Golang 官方库实现

在 Golang 1.8 版本之前， net/http 是没法提供方法的，所以当时开源社区涌现了不少第三方解决方案：[graceful](https://github.com/tylerstillwater/graceful) 、[grace](https://github.com/facebookarchive/grace)



其基本的思路差不多： 设计一个监听事件函数，监听事件结束后，通过 channel 等机制来等待主流程结束。



而在 1.8 版本之后。 net/http 引入了 server.Shutdown() 函数来进行优雅重启。

``` go
func (srv *Server) Shutdown(ctx context.Context) error {
	srv.inShutdown.setTrue()

	srv.mu.Lock()
	lnerr := srv.closeListenersLocked()
	srv.closeDoneChanLocked()
	for _, f := range srv.onShutdown {
		go f()
	}
	srv.mu.Unlock()

	pollIntervalBase := time.Millisecond
	nextPollInterval := func() time.Duration {
		// Add 10% jitter.
		interval := pollIntervalBase + time.Duration(rand.Intn(int(pollIntervalBase/10)))
		// Double and clamp for next time.
		pollIntervalBase *= 2
		if pollIntervalBase > shutdownPollIntervalMax {
			pollIntervalBase = shutdownPollIntervalMax
		}
		return interval
	}

	timer := time.NewTimer(nextPollInterval())
	defer timer.Stop()
	for {
		if srv.closeIdleConns() && srv.numListeners() == 0 {
			return lnerr
		}
		select {
		case <-ctx.Done():
			return ctx.Err()
		case <-timer.C:
			timer.Reset(nextPollInterval())
		}
	}
}
```

该函数可以优雅关闭服务，而不中断任何正在处理的活跃链接，关闭过程中会先关闭所有的 监听器，然后关闭所有的空闲链接。直到所有的链接关闭后退出。

如果有提供 Context 的话，有可能会提前关闭返回错误。



1. 在上面的代码中，会先通过 `srv.inShutdown.setTrue()` 将Server标记为正在关闭中。这是一个原子性的操作。

2. 随后关闭监听`srv.closeListenersLocked()`，不再处理新到来的请求。

3. 随后运行回调函数集合 `srv.onShutdown()` 
4.  `nextPollInterval` 设置了轮询时间，time.Sleep 是用阻塞当前 Goroutine 的方式来实现的，它需要调度先唤醒当前 Goroutine，才能唤醒后续的逻辑。**而 Ticker 创建了一个底层数据结构定时器 runtimeTimer，并且监听 runtimeTimer 计时结束后产生的信号。**这个 runtimeTimer 是 Golang 定义的定时器，做了一些比较复杂的优化。比如在有海量定时器的场景下，runtimeTimer 会为每个核，创建一个 runtimeTimer，进行统一调度，所以它的 CPU 消耗会远低于 time.Sleep。所以说，使用 ticker 是 Golang 中最优的定时写法。
5.  在 for 循环中，调用 `closeIdleConns` 一直等待链接空闲关闭，等完之后，利用步骤4的机制休眠 500ms 。如果完成，关闭连接，如果未完成，则跳过，等待下次循环。

``` go

// closeIdleConns 关闭所有的连接并且记录是否服务器的连接已经全部关闭
func (s *Server) closeIdleConns() bool {
  s.mu.Lock()
  defer s.mu.Unlock()
  quiescent := true
  for c := range s.activeConn {
    st, unixSec := c.getState()
    // Issue 22682: 这里预留5s以防止在第一次读取连接头部信息的时候超过5s
    if st == StateNew && unixSec < time.Now().Unix()-5 {
      st = StateIdle
    }
    if st != StateIdle || unixSec == 0 {
      // unixSec == 0 代表这个连接是非常新的连接，则标记位需要标记false
      quiescent = false
      continue
    }
    c.rwc.Close()
    delete(s.activeConn, c)
  }
  return quiescent
}
```

在 closeIdleConns 函数里，会遍历所有的请求，如果已经处理完操作处于 Idle 状态，就关闭链接，直到所有的链接都关闭，才返回。



所以如果想要实现优雅重启，在 main 函数所在的 Goroutine 中插入如下代码即可。

``` go
    // Wait for interrupt signal to gracefully shutdown the server with
	// a timeout of 5 seconds.
	quit := make(chan os.Signal, 1)
	// kill (no param) default send syscall.SIGTERM
	// kill -2 is syscall.SIGINT
	// kill -9 is syscall.SIGKILL but can't be caught, so don't need to add it
	signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
	<-quit
	log.Infof("Shutting down server...")

	if err := insecureServer.Shutdown(ctx); err != nil {
		log.Error("Insecure Server forced to shutdown:", log.Err(err))

		return err
	}
```



# 错误捕获 Recovery

前面已经介绍了 Context 的设计理念[^1] , 可以通过 Pipeline 的方式处理一个请求，来实现插入中间件的目的。

在上篇文章中可以采用 defer 、recover 函数兜住，c.Next 中抛出的异常，并且在 HTTP 请求中返回 500 内部错误的状态码。

``` go

// recovery 机制，将协程中的函数异常进行捕获
func Recovery() framework.ControllerHandler {
  // 使用函数回调
  return func(c *framework.Context) error {
    // 核心在增加这个 recover 机制，捕获 c.Next()出现的 panic
    defer func() {
      if err := recover(); err != nil {
        c.Json(500, err)
      }
    }()
    // 使用 next 执行具体的业务逻辑
    c.Next()

    return nil
  }
}
```

但是这个 Recovery 还是比较初级的实现。但是这个还是太初级， 和开源的 Gin 这种很完善的 Web Server 相比，**缺少底层连接的异常** 和 **异常堆栈打印** 的功能。 

{{< admonition tip "recovery 的局限性" true >}}
recovery 不能恢复子协程的崩溃。也就是说如果在 Context 中起一个新的协程并且新的协程发生 panic，会导致整个 Web 服务停止运行。
{{< /admonition >}}



``` go

return func(c *Context) {
    defer func() {
      if err := recover(); err != nil {
        // 判断是否是底层连接异常，如果是的话，则标记 brokenPipe
        var brokenPipe bool
        if ne, ok := err.(*net.OpError); ok {
          if se, ok := ne.Err.(*os.SyscallError); ok {
            if strings.Contains(strings.ToLower(se.Error()), "broken pipe") || strings.Contains(strings.ToLower(se.Error()), "connection reset by peer") {
              brokenPipe = true
            }
          }
        }
        ...

                
        if brokenPipe {
          // 如果有标记位，我们不能写任何的状态码
          c.Error(err.(error)) // nolint: errcheck
          c.Abort()
        } else {
          handle(c, err)
        }
      }
    }()
    c.Next()
  }
```



我们可以先判断底层抛出的异常是否是网络异常 (net.OpError) ，再根据系统提示的错误说明判断出错类型，来判断这个异常是否连接中断。如果是，就设置标记位。并且用 `c.Abort()` 来中断后续处理逻辑。









[^1]: [实现一个Web服务-(一)](https://kiosk007.top/post/%E5%AE%9E%E7%8E%B0%E4%B8%80%E4%B8%AAweb%E6%9C%8D%E5%8A%A1-%E4%B8%80/)