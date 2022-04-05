---
title: Golang 的内存对齐
author: kiosk
tags:
  - golang
categories:
  - programming
date: 2022-03-30 00:36:00
---

今天又在网上看到了一个以前不知道的概念，内存对齐？从名字就可以看出来，这是一个偏性能优化类的概念。

<!--more-->

如果使用了 go-lint，会有这样一个[检查项](maligned)。

```
linters-settings:
  maligned:
    # Print struct with more effective memory layout or not.
    # Default: false
    suggest-new: true
```

检查是否有更有效的内存布局，其本质就对 Golang 中 Struct 结构体字段的内存对齐进行检查。



## 前情概要

在分析内存对齐之前，我们先看看几个关于操作系统对内存是如何管理的。

内存是与CPU是程序运行的搭档，内存用于暂存CPU中的运算数据。早期，程序直接运行在物理内存上，直接操作物理内存，但是存在一些问题，比如使用效率低，地址空间不隔离等问题。就出现了虚拟内存这个概念，虚拟内存是程序和物理内存之间的一个中间层，在Liunx操作系统中。虚拟内存被划为`用户空间` 和 `内核空间`，用户只能访问用户空间的虚拟地址，而内核空间只有通过系统调用和外设中断才能访问。

CPU与内存的工作关系是：当执行一个程序时，硬盘上的程序被加载进内存，CPU对内存发起寻址操作，将加载到内存中的指令翻译出来，CPU通过总线寻址、取指令、执行取数据与寄存器交互，然后CPU运算，在输出数据至内存。



而CPU每次寻址能传输的数据大小和CPU位数有关，常见的CPU位数有8位、16位、32位、64位。位数越高每次执行的数据量就越大，性能也就越强。os的位数一般和CPU的位数相匹配， 32位CPU可以寻址4GB的内存空间，



## 什么是对齐

先看下面的这两个结构体

```go
type T1 struct {
	a int8
	b int64
	c int16
}

type T2 struct {
	a int8
	c int16
	b int64
}
```

在 64位的平台上 `T1` 占用 24 Bytes，`T2` 占用16 Bytes ；

在 32位的平台上 `T1`占用 16 Bytes，`T2`占用 12 Bytes 。



这是为什么呢？



因为编译器使用了内存对齐策略，所以结果就不一样了。而编译器要做对齐主要基于以下2个原因。

- 平台（移植性）

- 性能（若访问未对齐的内存，将导致CPU进行2次内存访问，并且要花费额外的时钟中期对齐预算。而本身就对齐的内存仅需要一次访问就可以一次访问完成读取动作，标准的空间换时间的做法）



每个特定平台上的编译器都有自己的默认"对齐系数"，常用平台默认对齐系数如下：

- 32位系统对齐系数是4
- 64位系统对齐系数是8



以上面的 `T1`、`T2` 来说，在 x86_64 平台上，`T1`的内存布局为：

![](https://img1.kiosk007.top/static/images/go/T1.png)

而T2的内存布局为

![](https://img1.kiosk007.top/static/images/go/T2.png)

从中可以看到T1存在了许多 padding，显然占据了不少空间，那么也就不难理解为什么改变结构体成员的位置可以达到缩小结构体占用大小的疑问了。



## 结构体的对齐规则

其实上面的例子已经可以看出对齐规则了，下面再简单总结下：

- 对于结构体的各个成员，第一个成员位于偏移为`0`的位置，结构体第一个成员的偏移量(offset)为`0`，以后每个成员相对于结构体首地址的`offset`都是该成员大小与有效对齐值中较小那个的整数倍，如有需要编译器会在成员之间加上填充字节。
- 除了结构成员需要对齐，结构本身也需要对齐，结构的长度必须是编译器默认的对齐长度和成员中最长类型中最小的数据大小的倍数对齐。



**举个例子**

```
// 64位平台，对齐参数是8
type User struct {
    A int32 // 4
    B []int32 // 24
    C string // 16
    D bool // 1
}

func main()  {
    var u User
    fmt.Println("u1 size is ",unsafe.Sizeof(u))
}
// 运行结果
u size is  56
```



- 第一个字段类型是`int32`，对齐值是4，大小为4，所以放在内存布局中的第一位.
- 第二个字段类型是`[]int32`，对齐值是8，大小为`24`，按照第一条规则，偏移量应该是成员大小`24`与对齐值`8`中较小那个的整数倍，那么偏移量就是`8`，所以`4-7`位会由编译进行填充，一般为`0`值，也称为空洞，第`9`到`32`位为第二个字段`B`.
- 第三个字段类型是`string`，对齐值是`8`，大小为`16`，所以他的内存偏移值必须是8的倍数，因为`user`前两个字段就已经排到了第`32`位，所以`offset`为`32`正好是`8`的倍数，不要填充，从`32`位到`48`位是第三个字段`C`.
- 第四个字段类型是`bool`，对齐值是`1`，大小为`1`，所以他的内存偏移值必须是`1`的倍数，因为`user`前两个字段就已经排到了第`48`位，所以下一位的偏移量正好是`48`，正好是字段`D`的对齐值的倍数，不用填充，可以直接排列到第四个字段，也就是从`48`到第`49`位是第三个字段`D`.

## 空结构体的字段对齐

`Go`语言中空结构体的大小为`0`，如果一个结构体中包含空结构体类型的字段时，通常是不需要进行内存对齐的，举个例子：

```go
type demo1 struct {
    a struct{}
    b int32
}

func main()  {
    fmt.Println(unsafe.Sizeof(demo1{}))
}
运行结果：
4
```

但是如果 `struct`在结构体的最后一个字段时，需要内存对齐，因为如果有内存指向该字段是，返回的的地址在结构体之外。所以当`struct{}`作为结构体成员中最后一个字段时，要填充额外的内存保证安全。

```go
type demo2 struct {
    a int32
    b struct{}
}

func main()  {
    fmt.Println(unsafe.Sizeof(demo2{}))
}
运行结果：
8
```



## go-lint 检查

在文章的开头有提到过使用 `golangci-lint` 命令行工具进行内存对齐检查。首先我们打印一下 `linters` 中和 `mem` 有关的选项，从后面的说明已经其功能了。

```
golangci-lint linters |rg "mem"
maligned [deprecated]: Tool to detect Go structs that would take less memory if their fields were sorted [fast: false, auto-fix: false]
```

还是上面最开始提到的例子。

```
package main

import "fmt"

type T1 struct {
   a int8
   b int64
   c int16
}

func main() {
   fmt.Println(&T1{})
}
```

运行lint检查命令。

```bash
$ golangci-lint run --no-config --disable-all -E maligned     
WARN [runner] The linter 'maligned' is deprecated (since v1.38.0) due to: The repository of the linter has been archived by the owner.  Replaced by govet 'fieldalignment'. 
main.go:5:9: struct of size 24 bytes could be of size 16 bytes (maligned)
type T1 struct {
        ^

```

可以看到 `lint` 报了一个 **WARN** ，`T1` 这个结构体的字节数可以更少的。



参考：

- [golang 内存对齐](https://xie.infoq.cn/article/594a7f54c639accb53796cfc7)
- [golang是否有必要内存对齐](https://ms2008.github.io/2019/08/01/golang-memory-alignment/)

- [[Golang]详解内存对齐](https://segmentfault.com/a/1190000040528007)

  




