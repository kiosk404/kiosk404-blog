# Golang 泛型入门

前言:
Go 1.18 版本正式推出了，而这个版本最大的亮点则是正式支持泛型。

<!--more-->



## 泛型编程

在之前的版本中`Go`语言想做一些通用的数据类型的编程操作的时候，可能大部分还是使用`interface`来进行编程。但是代码里面会出现各种断言操作。



## 准备工作

安装Go 1.18 版本

1. 使用下面的命令安装beta版本

``` bash
$ go install golang.org/dl/go1.18@latest
```

2. 运行如下命令来更新

```bash
 $ go1.18 download
```

3. 创建alias

```bash
$ alias go=go1.18
$ go version

```



## 初体验

如果没有泛型，我们必须对变量的类型进行强制定义，当然 Golang 中还有`intererface{}` 这个类型。假设我们需要写一个可以传入任意类型的排序函数，则需要在函数内判断类型，如下所示。



``` go
func bubbleSort(sequence []interface{}) {
	switch sequence[0].(type) {
	case int:
		for i := 0; i < len(sequence)-1; i++ {
			for j := 0; j < len(sequence)-i-1; j++ {
				if sequence[j].(int) > sequence[j+1].(int) {
					sequence[j], sequence[j+1] = sequence[j+1], sequence[j]
				}
			}
		}

	case float64:
		for i := 0; i < len(sequence)-1; i++ {
			for j := 0; j < len(sequence)-i-1; j++ {
				if sequence[j].(float64) > sequence[j+1].(float64) {
					sequence[j], sequence[j+1] = sequence[j+1], sequence[j]
				}
			}
		}
	}
}

func main() {
	sequence := []interface{}{12.12, 1.1, 99.4, 99.2, 88.8, 2.3}
	bubbleSort(sequence)
	fmt.Println(sequence)
}

```



下面我们看看泛型函数如何实现我们的需求。

`Go`支持泛型函数和泛型类型，首先我们看一下泛型函数，下面是一个标准的泛型函数标准模板：

```go
// GenericFunc 一个标准的泛型函数模板
func GenericFunc[T any](args T) {
	  // logic code
}
```

函数签名里面多了`[T any]`部分，这就是`Go`泛型的参数列表，其中`T`就是参数，`any`为参数的约束。



``` go
func bubbleSort[T int | float64](sequence []T) {
	for i := 0; i < len(sequence)-1; i++ {
		for j := 0; j < len(sequence)-i-1; j++ {
			if sequence[j] > sequence[j+1] {
				sequence[j], sequence[j+1] = sequence[j+1], sequence[j]
			}
		}
	}
}

func main() {
	sequence := []float64{2.12, 1.1, 99.4, 99.2, 88.8, 2.3}
	bubbleSort[float64](sequence)
	fmt.Println(sequence)
}

```



上面编写的泛型示例都是基于泛型函数进行的，但是我们有时候编程需要定义一些复合数据类型的数据结构，例如我们要实现一套缓存组件代码，缓存的内容`Value`可能有多重形式，一个缓存有`Get`方法有 `Set` 方法。一个写入缓存，一个取出缓存内容，如下的代码：

``` go
type Element interface {
	int | int64 | float64 | string
}

type Cache[V Element] struct {
	size  int
	value map[string]V
}
```

上面的代码我们就通过泛型编程定义一个`value`类型只能为`Element`类型集合的`Cache`结构，`Cache[V Element]`的中括号里面的就是泛型约束条件。

接着给Cache添加行为方法，方法签名上的`c *Cache[V]`就代表是一个泛型的`Cache`结构。

```go
type Element interface {
	int | int64 | float64 | string
}

type Cache[V Element] struct {
	size  int
	value map[string]V
}

func (c *Cache[V]) Set(key string, v V) {
	c.value[key] = v
	c.size = len(c.value)
}

func (c *Cache[V]) Get(key string) (ok bool, v V) {
	v, ok = c.value[key]
	return
}

func NewCache[V Element]() *Cache[V] {
	return &Cache[V]{
		size:  0,
		value: map[string]V{},
	}
}
```

使用就和函数泛型差不多，在中括号里面指定泛型类型：

```go
func main() {
    cache := NewCache[int64]()
	  cache.Set("a", 10)
	  _, v := cache.Get("a")
	  fmt.Println(v)
}
```



## 泛型和interface的区别

这么看来泛型和 `interface{}` 相比只是代码的可读性变的强大了吗？

其实不然。以上面的 ”缓存组件“ 为例

1.18 以下版本的Go为了支持任意类型，会使用空接口如下所示

``` go
type Cache struct {
  size   int 
  values map[string]interface{}
}

func (c *Cache) Set(k string, v interface{}) {
  c.values[k] = v
  c.size = len(c.values)
}

func (c *Cache) Get(k string) (v interface{}, ok bool) {
  v , ok = c.value[k]
  return 
}
```



一般在 `Set` 的时候没有什么不方便，因为到空接口的赋值是不需要额外的处理。但是 `Get` 的时候就需要将取出的数据进行一次类型断言。

我们知道空接口本质是一对指针。如果用来装载值类型的话，会发生逃逸到堆上。

![](https://img1.kiosk007.top/static/images/go/generic_interface1.png)

也就是说，在上面的cache中，value 是 `interface{}`空接口，而`interface{}`空接口是一个包含类型和值的的指针， `data`指针会指向堆上的 int 数据。那么就是老的实现方式占用了2倍的内存。

下来我们看看 泛型的实现。

``` go
func main() {
	cache := NewCache[int64]()
	cache.Set("a", 10)
	_, _ = cache.Get("a")
	t := reflect.TypeOf(cache.value)
	fmt.Println(t)       // map[string]int64
	fmt.Println(t.Elem().String())   // int64
}
```

通过反射可以看出 `cache.value` 底层的map本身值就是 `int64` , 没有额外的堆分配。也就是使用泛型的实现节省了程序在运行过程中的额外内存消耗的问题。



这样的实现本质也是有消耗的。其实 Go 编译器在编译阶段就为 int64 生成了一套代码，同样的 `string` 、`float64` 也生成了一套代码，这样会导致编译后的产物变大。



**所以泛型是编译阶段的代码生成，其不会代替空接口的地位，空接口主要来实现语言的动态性，他们的使用场景不一样**











参考:

- [Golang Generic Programming](https://blog.ibyte.me/post/golang-generic-programming/)
- [[Golang]泛型要来了吗？](https://www.bilibili.com/video/BV1PL41147Y5)

