# Golang
- [sample](/topic/interview/golang/#sample)
  - [new 和 make 的区别？](/golang?id=new-和-make的区别)
  - [Golang Slice](/golang?id=golang-slice)
  - [go中 import/const/var/init/main 的执行顺序](/)
  - [defer的执行顺序](/topic/interview/golang/#defer的执行顺序)
  - [什么是 go 的内存逃逸](golang?id=什么是-go-的内存逃逸)



## new 和 make的区别?
Go语言中 new 和 make 是两个内置函数，主要用来创建并分配类型的内存。

new 只分配内存，而 make 只能用于 slice、map 和 channel 的初始化。
new返回的是他们的指针，指针指向分配类型的内存地址。而make 返回的是他们类型的本身，因为chan、map、slice 本身就是引用类型，所以没有必要再返回他们的指针。

- new
用途：为值类型（像结构体、数组、基本数据类型等）分配内存，并且会把内存初始化为零值。
返回结果：返回一个指向新分配的零值的指针。

- make
用途：专门用于初始化切片（slice）、映射（map）和通道（channel）这三种引用类型。
返回结果：返回的是类型本身，并非指针。

## Golang Slice
底层是如何实现的？
答：切片本身并不是动态数组或者数组指针。它内部实现的数据结构通过指针引用底层数组

``` go
type slice struct {
  array unsafe.Pointer
  len   int
  cap   int
}
```

> 切片的结构体由3部分构成，Pointer 是指向一个数组的指针，len 代表当前切片的长度，cap 是当前切片的容量。cap 总是大于等于 len 的。
> 详见：https://halfrost.com/go_slice/

如何扩容
当前所需容量大于原先容量的2倍时，则申请当前所需的容量。
如果上述条件不满足，则进行如下判断
原切片长度小于1024则申请原先容量的2倍

> 否则每次增加 1/4 ，直到大于所需的容量为止。
> 详见：https://halfrost.com/go_slice/

判断 cap 的技巧
> 核心问题：如果append需要扩容，就会完全开辟一个新空间。
> 详见：https://coolshell.cn/articles/21128.html


*<br/>
*<br/>
*<br/>
*<br/>
*<br/>
*<br/>
*<br/>
*<br/>
*<br/>
*<br/>
*<br/>
*<br/>
*<br/>
*<br/>
*<br/>
*<br/>
*<br/>
*<br/>
*<br/>
*<br/>
*<br/>
*<br/>
*<br/>
*<br/>
*<br/>
*<br/>
*<br/>
*<br/>
*<br/>
*<br/>
*<br/>
*<br/>
*<br/>
*<br/>
*<br/>
*<br/>
*<br/>
*<br/>
*<br/>
*<br/>
*<br/>
*<br/>
*<br/>
*<br/>

### 什么是 go 的内存逃逸 