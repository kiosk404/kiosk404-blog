# Golang 内存逃逸分析

最近总是碰到有人讨论内存逃逸的问题，而且上升到装逼必备的地步，是时候了解一下什么叫内存逃逸了，于是在网上查了一下相关的资料。


<!--more-->

## 什么是内存逃逸 

初次看到这个话题，我是懵逼的，只知道内存逃逸很高级。还是值得理解一下的。

<br/>

**先看一个C++ 和 Go 的例子**。

```go
package main

func foo(arg_val int)(*int) {

    var foo_val int = 11;
    return &foo_val;
}

func main() {

    main_val := foo(666)

    println(*main_val)
}
```

编译运行

```go
$ go run main.go
11
```

竟然没有报错，并且返回了11。了解 C/C++ 的同学应该知道，这是不被允许的，因为外部函数用了子函数的局部变量。因为子函数的 `foo_val` 会被销毁。如下：

```c
#include <stdio.h>

int *foo(int arg_val) {

    int foo_val = 11;

    return &foo_val;
}

int main()
{
    int *main_val = foo(666);

    printf("%d\n", *main_val);
}

```

编译：

```
$ gcc main.c 
main.c: In function ‘foo’:
main.c:7:12: warning: function returns address of local variable [-Wreturn-local-addr]
     return &foo_val;
            ^~~~~~~~
```

出了一个警告，不管他，再运行

```
$ ./a.out 
段错误 (核心已转储)
```

程序崩溃.

如上C/C++编译器明确给出了警告，foo把一个局部变量的地址返回了





我们知道程序运行是需要内存的，而内存空间包含两个最重要的区域：堆区(Heap)和栈区(Stack)。在C语言中，栈区存放函数参数，局部变量，堆也被称为动态内存分配，**它是由程序员手动完成申请和释放的**。

<br/>

C 语言中，函数不能返回局部变量**地址**（指存放在栈区的局部变量地址），这是因为函数调用完毕之后，局部变量会随函数一起被释放掉。其地址指向的内容可能没变也可能被改变。所以想要返回函数局部变量的地址，必须是动态分配的，也就是堆上的数据。



但是Go是一门自带GC的语言，真正解放了程序员，不需要程序员手动管理内存，内存管理交给了编译器，编译器会经过逃逸分析把变量合理的分配到"正确"的地方。



所以内存逃逸可以理解如下：

在一段程序中，每一个函数都会有自己的内存区域存放自己的局部变量、返回地址等，这些内存会由编译器在栈中进行分配，每一个函数都会分配一个栈桢，在函数运行结束后进行销毁，但是有些变量我们想在函数运行结束后仍然使用它，那么就需要把这个变量在堆上分配，这种从"栈"上逃逸到"堆"上的现象就成为内存逃逸。



## Go 内存分配

Go 语言的编译期会自动决定把一个变量放在栈还是堆上，编译器会做**逃逸分析（escape analysis）**，**当发现变量的作用域没有跑出函数范围**，**就可以在栈上**，**反之则必须分配在堆**。

<br/>

Go 是自带GC的，程序员可以随意在函数内返回局部变量的地址，而不用关心内存的释放问题。Go 的编译器会自行决定变量分配在堆上还是分配在栈上。一般来说Go编译器将为该函数的栈中的函数分配局部变量。但是，如果编译器在函数返回后无法证明变量未被引用，则编译器必须在垃圾收集堆上分配变量以避免悬空指针错误。

``` go
func GetWares() *Wares {
  var budget int 
  budget = dbFactory.GetBudget()
  if budget > 100 {
    reutrn &Wares{name: "篮球"}
  } else if budget > 50 {
    return &Wares{name: "足球"}
  }
  return nil
}
```



上面简单的例子可以看出局部变量 `budget` 只在函数内有使用，所以budget是分配在栈上。

`&Wares` 被分配到了堆上。



## 什么是逃逸分析

在编译程序优化理论中，逃逸分析是一种确定指针动态范围的方法，简单来说就是分析在程序的哪些地方可以访问到该指针

通俗地讲，逃逸分析就是确定一个变量要放堆上还是栈上，规则如下：

1. 是否有在其他地方（非局部）被引用。只要**有可能**被引用了，那么它**一定**分配到堆上。否则分配到栈上
2. 即使没有被外部引用，但对象过大，无法存放在栈区上。依然有可能分配到堆上



## 为什么要逃逸呢？

这个问题其实可以反过来想，如果变量全部分配到堆上会发生甚么样的情况呢？

- 垃圾回收（GC）的压力不断的变大
- 申请、分配、回收内存的开销增大
- 动态分配内存产生很多碎片

总的来说，频繁的再堆上申请、分配内存是有代价的。会影响程序的运行效率。间接的影响到整个系统。



## 如何分析逃逸呢？

第一：通过编译器的命令，就可以看到详细的逃逸过程。指令集`-gcflags`用于将标识参数传递给Go编译器，涉及如下：

- `-m` 会打印出逃逸分析的优化策略，可以使用4个`-m`, 实际上用一个就可以了
- `-l` 会禁用函数内联，禁掉inline可以更好观察逃逸情况，减少干扰

``` bash
$ go build -gcflags '-m -l' main.go
```



第二：反编译

``` bash
$ go tool compile -S main.go
```





## 几个逃逸分析的例子

**逃逸案例**

- 案例一：指针

```go
type User struct {
    ID     int64
    Name   string
}

func GetUserInfo() *User {
    return &User{ID: 394244, Name: "Kiosk007"}
}

func main() {
    _ = GetUserInfo()
}
```

执行如下命令：

```bash
$ go build -gcflags '-m -l' main.go
# command-line-arguments
./main.go:10:54: &User literal escapes to heap
```

通过分析可知，`&User` 逃逸到了堆上，这是因为 `GetUserInfo()` 返回的是一个指针对象，引用返回到了方法之外，因此编译器会将该对象分配到堆上， 而不是栈上。



- 案例二： 未确定类型

```go
func main() {
    str := "Kiosk007"
    fmt.Println(str)
}
```

执行如下命令：

```bash
$ go build -gcflags '-m -l' main.go
# command-line-arguments
./main.go:8:13: ... argument does not escape
./main.go:8:13: str escapes to heap
```

通过分析可知 `str` 变量逃逸到了堆上，也就是该对象在堆上分配，这是因为当形参是 `internal`类型时，在编译阶段编译器无法确定其类型，因此会产生逃逸



- 案例三：泄露参数

``` go
type User struct {
    ID     int64
    Name   string
}

func GetUserInfo(u *User) *User {
    return u
}

func main() {
    _ = GetUserInfo(&User{ID: 394244, Name: "Kiosk007"})
}
```

执行如下命令：

``` bash
$ go build -gcflags '-m -l' main.go
# command-line-arguments
./main.go:9:18: leaking param: u to result ~r1 level=0
./main.go:14:63: main &User literal does not escape
```

上述代码说明变量 `u`是一个泄露的参数，结合代码可得知其传给 `GetUserInfo` 方法后，没有做任何引用之类的涉及变量的动作，直接就把这个变量返回出去了。因此这个变量实际上并没有逃逸，





## 总结

- 逃逸分析在编译阶段确定哪些变量可以分配在栈中，哪些变量分配在堆上
- 逃逸分析减轻了`GC`压力，提高程序的运行速度
- 栈上内存使用完毕不需要`GC`处理，堆上内存使用完毕会交给`GC`处理
- 函数传参时对于需要修改原对象值，或占用内存比较大的结构体，选择传指针。对于只读的占用内存较小的结构体，直接传值能够获得更好的性能
- 根据代码具体分析，尽量减少逃逸代码，减轻`GC`压力，提高性能



参考：

[我要在栈上。不，你应该在堆上](https://studygolang.com/articles/20593)

[详解Go语言中的内存逃逸](https://segmentfault.com/a/1190000040450335)











