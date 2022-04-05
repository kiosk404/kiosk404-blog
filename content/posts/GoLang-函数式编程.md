---
title: GoLang 编程技巧 (一)
author: kiosk
tags:
  - golang
categories:
  - programming
date: 2021-01-30 11:12:00
---
本文为了巩固一下 Go 的基础，主要涉及 Slice、Interface、函数式编程等。

<!--more-->


也是读了<a href="https://coolshell.cn/" style="">左耳朵耗子</a> 叔的 [Go编程模式](https://coolshell.cn/articles/21128.html) 的系列文章发现有的细节确实之前也有遗漏，刚好也趁机复习巩固一下。

# 基本结构
## Slice

Slice，中文翻译叫“切片”，这个东西在Go语言中不是数组，而是一个结构体，其定义如下：

``` golang

type slice struct {
    array unsafe.Pointer //指向存放数据的数组指针
    len   int            //长度有多大
    cap   int            //容量有多大
}
```

Slicce标头的**array**字段是底层真正指向数组的指针。

![](https://i0.wp.com/golangbyexample.com/wp-content/uploads/2020/05/slice.jpg?w=391&ssl=1)

Golang 的切片是子集。切片可以是数组、列表或字符串的子集。可以从一个字符串中提取多个片段，每个片段作为一个新变量。

### **与数组的不同**：
数组在声明为一定大小后，不能调整大小，而切片可以调整大小。切片是引用类型，而数组是值类型。

### **在Golang中创建切片**

``` golang
func main() {
	var stringSlice = []string{"This", "is", "a", "string", "slice"}
	fmt.Println(stringSlice)  // prints [This is a string slice]
	// res:  [This is a string slice]

	myset := []int{0,1,2,3,4,5,6,7,8}

	/* take slice */
	s1 := myset[0:4]
	fmt.Println(s1)
	// res:  [0 1 2 3]

	mystring := "Go programming"

	/* take slice */
	s2 := mystring[0:2]
	fmt.Println(s2)
	// res:  Go

	numbers := make([]int, 3, 5)
	fmt.Printf("numbers=%v\n", numbers)
	fmt.Printf("length=%d\n", len(numbers))
	fmt.Printf("capacity=%d\n", cap(numbers))
	// res:
	//numbers=[0 0 0]
	//length=3
	//capacity=5
}
```

### **切片引用**

切片是引用类型，那么就意味着数组指针的问题——数据会发生共享！下面我们来看看 Slice 的一些操作：

``` golang

foo = make([]int, 5)
foo[3] = 42
foo[4] = 100

bar  := foo[1:4]
bar[1] = 99

// 打印 foo
// 打印 bar
```
1. 首先，创建一个 foo 的 Slice，其中的长度和容量都是 5；
2. 然后，开始对 foo 所指向的数组中的索引为 3 和 4 的元素进行赋值；
3. 最后，对 foo 做切片后赋值给 bar，再修改 bar[1]。

最终的foo和bar的结果是什么呢? 是不是和想象的不太一样，这是因为切片操作的底层数组是同一个数组。foo 和 bar 的内存是共享的，所以，foo 和 bar 对数组内容的修改都会影响到对方。
```
fmt.Println(foo)  // res: [0 0 99 42 100]
fmt.Println(bar)  // res: [0 99 42]
```


再来看一个 `append` 的例子。
``` golang
a := make([]int, 32)
b := a[1:16]
a = append(a, 1)
a[2] = 42

// 打印a
// 打印b
```
在这段代码中，把 `a[1:16]` 的切片赋给 b ，此时，a 和 b 的内存空间是共享的，然后，对 a 做了一个 append()的操作，这个操作会让 a 重新分配内存，这就会导致 a 和 b 不再共享，如下图所示：
<img src="https://static001.geekbang.org/resource/image/9a/13/9a29d71d309616f6092f6bea23f30013.png" style="max-width: 70%;border-radius: 6px">

这时 a 和 b 的值是多少？append()操作让 a 的容量变成了 64，而长度是 33。这里你需要重点注意一下，**append()这个函数在 cap 不够用的时候，就会重新分配内存以扩大容量，如果够用，就不会重新分配内存了！**
``` 
fmt.Println(a)  // res: [0 0 42 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 1]
fmt.Println(b)  // res: [0 0 0 0 0 0 0 0 0 0 0 0 0 0 0]

```


那既然这样相同的例子我们再来一遍，如果让a不要重新分配内存(比如初始化a的时候使用` a := make([]int, 33))`，那么b的结果就会变成 `[0 42 0 0 0 0 0 0 0 0 0 0 0 0 0]` **注意**：这时的b会因为`a[2]` 的变化而变化。

同样的例子如下，只要没有发生
``` golang
func main() {
	path := []byte("AAAA/BBBBBBBBB")
	sepIndex := bytes.IndexByte(path,'/')

	dir1 := path[:sepIndex]
	dir2 := path[sepIndex+1:]

	fmt.Printf("len: %d cap: %d ",len(dir1),cap(dir1)) // prints: len: 4 cap: 14
	fmt.Printf("len: %d cap: %d ",len(dir2),cap(dir2)) // prints: len: 9 cap: 14

	fmt.Println("dir1 =>",string(dir1)) //prints: dir1 => AAAA
	fmt.Println("dir2 =>",string(dir2)) //prints: dir2 => BBBBBBBBB

	dir1 = append(dir1,"suffix"...)

	fmt.Println("dir1 =>",string(dir1)) //prints: dir1 => AAAAsuffix
	fmt.Println("dir2 =>",string(dir2)) //prints: dir2 => uffixBBBB
}
```

在这个例子中，dir1 和 dir2 共享内存，虽然 dir1 有一个 append() 操作，但是因为 cap 足够，于是数据扩展到了dir2 的空间。下面是相关的图示（注意上图中 dir1 和 dir2 结构体中的 cap 和 len 的变化）：

<img src="https://static001.geekbang.org/resource/image/17/aa/1727ca49dfe2e6a73627a52a899535aa.png" style="max-width: 70%;border-radius: 6px">


这里的 `dir1:=path[:sepIndex]` 没有触发重新分配内存，如果想要强行重新分配内存的话可以使用`dir1 := path[:sepIndex:sepIndex]` 最后一个参数叫“Limited Capacity”，于是，后续的 append() 操作会导致重新分配内存。




## Interface

Interface 接口是一个抽象概念，它支持Go中的**多态**。该接口的变量可以保存实现该类型的值。类型断言用于获取底层的具体值。接口也是给Go语言带来了无限扩展空间。其中 `io.Reader` 接口就是一个典型的例子，**io.Reader** 表示读取设备数据流的能力，可以从[网络、文件、字符串](https://golang.cafe/blog/golang-reader-example.html)等等。先简单介绍下 `io.Reader` 接口 ，后面会介绍如何使用接口式编程的方式封装 Reader 。

### 接口式编程

[io.Reader](https://golang.org/pkg/io/#Reader) interface 可以表示从实体中读取字节流。

``` golang
type Reader interface {
        Read(buf []byte) (n int, err error)
}
```

即只要实现了 `Read(buf []byte) (n int, err error)` 方法便就是 `io.Reader` 接口。Read最多将 `len(buf)` 字节读入buf并返回读取的字节, 直到读到 `io.EOF` 时返回。标准库中实现了很多Reader的实现。并且很多应用程序都接受 `Reader` 作为输入。


- **直接从字节流中读取**

这里分为 `Read`、 `io.ReadFull`、 `ioutil.ReadAll ` 三种方法。每种方法都有一些区别。

 1. 直接使用 Read 方法

``` golang
r := strings.NewReader("abcde")

buf := make([]byte, 4)
for {
	n, err := r.Read(buf)
	fmt.Println(n, err, buf[:n])
	if err == io.EOF {
		break
	}
}
```
在这个示例中，我们创建了一个字节流 `r`, 在循环从r中读取出数据。循环会执行3次，第一次读取4个字节，第二次读取1个字节，第三次读到 `io.EOF` 返回跳出循环。注意，Read方法读取时会清空 `buf` 里的数据，所以这里需要每次读完打印一下。再次读时，`buf` 里的数据会被重新覆盖。


```
4 <nil> [97 98 99 100]
1 <nil> [101]
0 EOF []
```

另外还可以使用 `io.ReadFull` 或者 `ioutil.ReadAll` 取读取字节流,`io.ReadFull`用法和`Read`差不多，`ioutil.ReadAll`不需要设置buf可直接返回buf。更多可参考：[How to use the io.Reader interface
](https://yourbasic.org/golang/io-reader-interface-explained/)

- **利用接口特性**

下面的代码是一个实时统计标准输入字符个数的代码。用户每次按下回车都可以看到当前输入的字符以及历史上已经输入的字符的个数。


``` golang
func CountNumber(input chan []byte) {
	count := 0
	for data := range input {
		count += len(data)
		fmt.Printf("Received %d Characters: === %s \n",count, data)
	}
}

func main() {
 	bytes := make(chan []byte)
 	fmt.Println("Please enter the string to be calculated：")
 	go CountNumber(bytes)
 	for {
    	var input string
		_, _ = fmt.Scanln(&input)
		bytes <- []byte(input)
  	}
}
```

代码很简单，但是似乎和我们要讲到的接口式编程没什么关系。下面我们用接口封装一下。

``` golang
func NewCountReader() *CountReader {
	return &CountReader{
		bytes: make(chan []byte),
		data:  nil,
	}
}

type CountReader struct {     // 声明CountReader对象
	bytes    chan []byte
	data     []byte
}

func (h *CountReader) Read(p []byte) (int, error) { // 实现Read方法
	ok := true
	for ok && len(h.data) == 0 {
		h.data, ok = <-h.bytes   // 将bytes里的数据全部传给 data
	}
	if !ok || len(h.data) == 0 {
		return 0, io.EOF    // 可能读到了结尾
	}

	l := copy(p, h.data)
	h.data = h.data[l:]
	return l, nil
}

func (h *CountReader) run() {
	b := bufio.NewReader(h)
	count := 0
	for true {

		buf := make([]byte, 4)
		n,err := b.Read(buf)
		if err == io.EOF {
			continue
		}
		count += len(buf[:n])

		fmt.Printf("Received %d Characters: === %s \n",count, buf)
	}
}
```

首先声明了一个结构体 `CountReader`, 再实现了一个 `Read()` 方法调用，我们知道实现了`Read()`即可以成为 `io.Reader` 接口的实现。也就是说 `CountReader` 就是一个 `io.Reader` ，那么 `io.Reader` 可以使用的方法，也可以给 `CountReader` 使用。这时就可以使用 `bufio` 这个库了。使用 `bufio.NewReader` 的函数对输入数据进行读取和计算。


<p class='div-border yellow'><code>bufio.NewReader()</code> 方法提供一个缓存buf, 默认缓存4k buffer 缓冲器的实现原理就是，将文件读取进缓冲（内存）之中，再次读取的时候就可以避免文件系统的 I/O 从而提高速度。同理在进行写操作时，先把文件写入缓冲（内存），然后由缓冲写入文件系统。</p>

``` go
func main() {
	fmt.Println("Please enter the string to be calculated：")
	Counter := NewCountReader()
	go Counter.run()
	for {
		var input string
		_, _ = fmt.Scanln(&input)
		Counter.bytes <- []byte(input)
	}
}

```
接下来通过上述代码即可完成相同的操作，这只是一个简单的例子，如果换成文件io、网络io就会有非常可观的收益。带来业务性能的提升。

# Functional Option

Functional Options 这个编程模式是一个函数式编程的应用案例，编程技巧也很好，是目前 Go 语言中最流行的一种编程模式。

假设实际编程中需要针对业务对象设置很多属性。

``` go
type Server struct {
    Addr     string
    Port     int
    Protocol string
    Timeout  time.Duration
    MaxConns int
    TLS      *tls.Config
}
```

在这个 Server 对象中，我们可以看到：

- 要设置侦听的 IP 地址 Addr 和端口号 Port。（必填）
- 协议、超时时间、最大链接数、TLS选项等属性需要配置。（非必填）

那么如何让调用方实现这个必填参数和非必填参数呢？一个方法是将非必填参数设成 `...interface{}` 但这样肯定不好，因为不同的参数类型都不一样。另一种方式就是将 必填参数和非必填参数分开了。

如非必填参数搞成一个结构体

``` golang
type Config struct {
    Protocol string
    Timeout  time.Duration
    Maxconns int
    TLS      *tls.Config
}
```

必填参数和这个 `Config` 直接传给初始化函数，如果没有要填的参数可以将 `Config` 设为 `nil` 。

这样一来 `Server` 结构体便成了这样, 初始化

``` golang
type Server struct {
    Addr string
    Port int
    Conf *Config
}


func NewServer(addr string, port int, conf *Config) (*Server, error) {
    //...
}

//Using the default configuratrion
srv1, _ := NewServer("localhost", 9000, nil) 

conf := ServerConfig{Protocol:"tcp", Timeout: 60*time.Duration}
srv2, _ := NewServer("locahost", 9000, &conf)
```

这样便已经是大多数人的作法了。但是不是没有修改空间，下面介绍一下 Functional Option 方式。

## **初始化 Server 示例**

首先我们定义一个 `Option` 类型:

``` golang
type Option func(*Server)
```

用函数式方式定义一组函数。

``` golang
func Protocol(p string) Option {
    return func(s *Server) {
        s.Protocol = p
    }
}
func Timeout(timeout time.Duration) Option {
    return func(s *Server) {
        s.Timeout = timeout
    }
}
func MaxConns(maxconns int) Option {
    return func(s *Server) {
        s.MaxConns = maxconns
    }
}
func TLS(tls *tls.Config) Option {
    return func(s *Server) {
        s.TLS = tls
    }
}
```

这组代码的含义是传入一个参数，返回一个函数，函数会将 `Server` 结构的对应参数值进行设置。例如，当我们调用其中的一个函数 MaxConns(30) 时，其返回值是一个 func(s* Server) { s.MaxConns = 30 } 的函数。

这下，我们可以定义一个 `NewServer` 函数，其中有一个可变参数 `option` ,用一个循环来设置 Server 的属性。不仅提供了默认值，还提供将默认值改成可修改选项进行修改。

```golang
func NewServer(addr string, port int, options ...func(*Server)) (*Server, error) {

  srv := Server{
    Addr:     addr,
    Port:     port,
    Protocol: "tcp",
    Timeout:  30 * time.Second,
    MaxConns: 1000,
    TLS:      nil,
  }
  for _, option := range options {
    option(&srv)
  }
  //...
  return &srv, nil
}
```

于是，我们在创建 Server 对象的时候，就可以像下面这样：

``` golang 

s1, _ := NewServer("localhost", 1024)
s2, _ := NewServer("localhost", 2048, Protocol("udp"))
s3, _ := NewServer("0.0.0.0", 8080, Timeout(300*time.Second), MaxConns(1000))
```

这下对 Server 的封装就像搭积木一样简单容易并且可视化很好。








