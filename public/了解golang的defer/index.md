# 了解golang的defer


golang 中有一个关键字 `defer` ，golang 的 defer 语句用于延迟调用。defer 会在当前函数返回之前执行 defer 注册的函数。

在 defer 语句所在的函数退出之前调用。defer 可以代替其它语言中 try…catch… 语句，也可以用来处理释放资源等收尾操作，比如关闭文件句柄、关闭数据库连接等。defer 还能用于 panic 的 recovery。

<!--more-->

下面了解 Golang 的一些具体知识点。

<br/>

# defer 的执行顺序

多个defer出现的时候，**它是一个栈的关系，先进后出**，也就是写在前面的 defer 调用的最晚。

```go
package main

import "fmt"

func main() {
    defer func1()
    defer func2()
    defer func3()
}

func func1() {
    fmt.Println("A")
}

func func2() {
    fmt.Println("B")
}

func func3() {
    fmt.Println("C")
}

// 执行效果
C
B
A
```

<br/>

# defer 与 return 的顺序

```go
func deferFunc() int {
	fmt.Println("defer func called")
	return 0
}

func returnFunc() int {
	fmt.Println("return func called")
	return 0
}

func returnAndDefer() int {

	defer deferFunc()

	return returnFunc()
}

func main() {
	returnAndDefer()
}

// 执行结果 
return func called
defer func called
```

结论是：先reture再调用 defer.

<br/>

# 有名函数返回值遇见defer的情况

在没有defer的情况下，其函数的返回值就是与return一致，但是有了defer就不一样了。因为是先return 再 defer，所以在执行完return之后，还要再执行defer里的语句，依然可以修改本应该返回的结果。

```go
func returnButDefer() (t int) {  //t初始化0， 并且作用域为该函数全域

    defer func() {
        t = t * 10
    }()

    return 1
}

func main() {
    fmt.Println(returnButDefer())
}

// 执行结果
10 
```

该`returnButDefer()`本应的返回值是`1`，但是在return之后，又被defer的匿名func函数执行，所以`t=t*10`被执行，最后`returnButDefer()`返回给上层`main()`的结果为`10`

<br/>

# defer 与 panic

遇到panic时，遍历本协程的defer链表，并执行defer。在执行 defer 过程中，遇到 recover 则停止 panic，返回recover处继续往下执行。如果没有遇到recover，遍历完本协程的defer链表后，向stderr 抛出panic信息。

第一种：

```go
func main() {
    defer_call()

    fmt.Println("main 正常结束")
}

func defer_call() {
    defer func() { fmt.Println("defer: panic 之前1") }()
    defer func() { fmt.Println("defer: panic 之前2") }()

    panic("异常内容")  //触发defer出栈

	defer func() { fmt.Println("defer: panic 之后，永远执行不到") }()
}

// 执行结果
defer: panic 之前2
defer: panic 之前1
panic: 异常内容
//... 异常堆栈信息
```

第二种：

```go
func main() {
    defer_call()

    fmt.Println("main 正常结束")
}

func defer_call() {

    defer func() {
        fmt.Println("defer: panic 之前1, 捕获异常")
        if err := recover(); err != nil {
            fmt.Println(err)
        }
    }()

    defer func() { fmt.Println("defer: panic 之前2, 不捕获") }()

    panic("异常内容")  //触发defer出栈

	defer func() { fmt.Println("defer: panic 之后, 永远执行不到") }()
}

// 执行结果
defer: panic 之前2, 不捕获
defer: panic 之前1, 捕获异常
异常内容
main 正常结束
```

<br/>

# defer 中包含panic

```go
func main()  {

    defer func() {
       if err := recover(); err != nil{
           fmt.Println(err)
       }else {
           fmt.Println("fatal")
       }
    }()

    defer func() {
        panic("defer panic")
    }()

    panic("panic")
}

// 执行结果
defer panic
```

触发 `panic("panic")` 后defer顺序出栈执行，第一个被执行的defer中会有 `panic("defer panic")` 异常语句，这个异常会覆盖掉main中的异常 `panic("panic")`



# defer 下的函数参数包含子函数

```go
func function(index int, value int) int {

    fmt.Println(index)

    return index
}

func main() {
    defer function(1, function(3, 0))
    defer function(2, function(4, 0))
}
```



这里有4个函数，他们的index序号分别是 1，2，3，4。那么这4个函数。分析输出结果时，先压栈1,再压栈2。因为要计算function1就必须计算3，所以function 3 先被执行，接下来执行 function 4，所以输出4.

```
// 输出结果
3
4
2
1
```


