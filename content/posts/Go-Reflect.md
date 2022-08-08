---
title: 'Go Reflect '
author: kiosk
tags:
  - golang
categories:
  - programming
date: 2021-01-18 23:57:00
---
Go 语言标准库 [reflect](https://golang.org/pkg/reflect/) 提供了运行时动态获取对象的类型和值以及动态创建对象的能力。反射可以帮助抽象和简化代码，提高开发效率。

<!--more-->

# 类型和接口 (Types and interfaces)

因为反射是建立在类型之上的，所以想要了解反射必须先知道Go 语言中所有的变量都有一个静态类型。例如 `int`、`[]byte`、`float32`、`*MyType`等等。
如下：
``` go
type MyInt int

var i int
var j MyInt
```

接口是一种特殊的类型，它表示固定的方法集。接口变量可以存储任何具体（非接口）值,只要该值实现接口的方法。最经典的例子便是 io.Reader \ io.Writer 。
``` go
// Reader is the interface that wraps the basic Read method.
type Reader interface {
    Read(p []byte) (n int, err error)
}

// Writer is the interface that wraps the basic Write method.
type Writer interface {
    Write(p []byte) (n int, err error)
}
```

有人说Go的接口是动态类型的，但这是误导。它们是静态类型。一个特殊的例子是空接口，即 `interface{}`。

在这个例子中。<font style="color:#00008B">os.OpenFile</font> 的返回参数tty的类型是 `*os.File`，由于 `*os.File` 实现了 <font style="color:#00008B">Read()</font> 方法，所以该类型可以被赋于类型 io.Reader (io.Reader是一个interface)。尽管 <code> *os.File</code> 实现了很多方法，但是变量r仅有一个方法Read。但内部的值仍包含有关该值的所有类型信息。这就是为什么我们可以做 `w=r.(io.Writer)` 的原因。

``` go
func main() {
	var r io.Reader
	tty, err := os.OpenFile("/dev/tty", os.O_RDWR, 0) // tty *os.File
	if err != nil {  return }
	r = tty

	var w io.Writer    
	w = r.(io.Writer)  // 由于r实际内部是有Write方法的，所以r可以被断言成 io.Writer

	if _, err = w.Write([]byte("12345"));err != nil { return }

	_ = tty.Close()
}
```

甚至可以继续执行下面的操作。
``` go
	var empty interface{}
	empty = w
	fmt.Println(reflect.TypeOf(empty))
```

## 反射基本用法
### <font style="color:#483D8B">**从接口值到反射对象**</font>

ValueOf用来获取输入参数接口中的数据的值。如果是空接口则返回 invalid 。
TypeOf用来动态获取输入参数接口中的值的类型，如果空接口则返回nil


``` go
	var num = 1.2345
	fmt.Println("type: ",reflect.TypeOf(num))     // type:  float64
	fmt.Println("value: ",reflect.ValueOf(num))   // value:  1.2345

```

也可以判断类型

``` go
func checkType(v interface{}) {
	t := reflect.TypeOf(v)
	switch t.Kind() {
	case reflect.Float64:
		fmt.Println("float")
	case reflect.Int:
		fmt.Println("int")
	case reflect.String:
		fmt.Println("string")
	default:
		fmt.Println("none")
	}

	switch v.(type) {   // 方法二
	case float64:
		fmt.Println("float")
	case int:
		fmt.Println("int")
	case string:
		fmt.Println("string")
	default:
		fmt.Println("none")
	}
}
```

### <font style="color:#483D8B">**从反射对象到接口值**</font>


go 提供了反射和反射的逆，可以通过 `.(type)` 断言的方式将一个Interface()转成他真正的类型。如果断言的类型不匹配，会发生panic。

可以通过下面几种方法从反射值对象 reflect.Value 中获取原值，如下表所示。

|方法名	|说 明|
| -- | -- |
| Interface() interface{} |	将值以 interface{} 类型返回，可以通过类型断言转换为指定类型 |
|Int() int64	 | 将值以 int 类型返回，所有有符号整型均可以此方式返回 |
|Uint() uint64	 | 将值以 uint 类型返回，所有无符号整型均可以此方式返回 |
|Float() float64 |	将值以双精度（float64）类型返回，所有浮点数（float32、float64）均可以此方式返回 |
|Bool() bool	 | 将值以 bool 类型返回 |
|Bytes() []bytes |	将值以字节数组 []bytes 类型返回 |
|String() string |	将值以字符串类型返回 |

``` go
	var pi float64
	pi = 3.1415926
	v := reflect.ValueOf(pi)
	y := v.Interface().(float64) // y will have type float64.
	fmt.Println(y)
```

### <font style="color:#483D8B">**要修改反射对象，该值必须可设置**</font>

如果运行下面的代码将会 <font style="color:#8B0000"> panic </font> ，这时因为 v 是不可设置的。

|方法名|	备  注|
| --  | -- |
|Elem() Value |	取值指向的元素值，类似于语言层*操作 |
|Addr() Value | 对可寻址的值返回其地址，类似于语言层&操作。当值不可寻址时发生宕机 |
|CanAddr() bool |	表示值是否可寻址|
|CanSet() bool |	返回值能否被修改。要求值可寻址且是导出的字段 |

``` go
var x float64 = 3.4
v := reflect.ValueOf(x)
v.SetFloat(7.1) // Error: will panic.
```

通过 `CanSet()` 即可判断，`CanSet()` 报告 v 的值是否可以改变。值只能在可寻址的情况下更改，并且不能通过使用未导出的结构字段获取。如果 CanSet 返回 false ，则调用 Set 或任何类型特定的 setter （例如 SetBool ，SetInt ）将会发生panic。
``` go
var x float64 = 3.4
v := reflect.ValueOf(x)
fmt.Println("settability of v:", v.CanSet())
```

v 不可寻址，因为 v 只是 x 的拷贝，即便把 x 换成 &x,还是不可寻址，因为 `reflect.ValueOf(&x)` 也仅仅是 x 指针的拷贝。实际上，所有通过 `reflect.ValueOf(x)` 返回的 reflect.Value 都是不可取地址的。但是通过调用 `reflect.ValueOf(&x).Elem()`，来获取任意变量x对应的可取地址的 Value

``` go
	var x float64
	x = 3.1415
	pv := reflect.ValueOf(&x)
	pv = pv.Elem()
	pv.SetFloat(7.1) 

	fmt.Println(x)   // 7.1
```



## 结构 Struct

| 方法                                                         | 说明                                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| Field(i int) StructField                                     | 根据索引，返回索引对应的结构体字段的信息。当值不是结构体或索引超界时发生panic |
| NumField() int                                               | 返回结构体成员字段数量。当类型不是结构体或索引超界时发生panic |
| FieldByName(name string) (StructField, bool)                 | 根据给定字符串返回字符串对应的结构体字段的信息。没有找到时 bool 返回 false，当类型不是结构体或索引超界时发生 panic |
| FieldByIndex(index []int) StructField                        | 多层成员访问时，根据 []int 提供的每个结构体的字段索引，返回字段的信息。没有找到时返回零值。当类型不是结构体或索引超界时发生 panic |
| FieldByNameFunc( match func(string) bool) (StructField,bool) | 根据匹配函数匹配需要的字段。当值不是结构体或索引超界时发生panic |





### 普通结构体
1. 先获取interface的reflect.Type，然后通过NumField进行遍历
2. 再通过reflect.Type的Field获取其Field
3. 最后通过Field的Interface()得到对应的value

假设有以下 `User struct`

``` go
type User struct {
	Id      int
	Name    string
	Work    Worker
}

type Worker struct {
	Id           int
	Occupation   string
}

func (u User) ReflectCallFunc() {
	fmt.Printf("My Id :%d ,My Name :%s ,My Occupation :%s",u.Id,u.Name,u.Work.Occupation)
}

func (u User) ReflectCallFuncHasArgs(foo int) {
	fmt.Printf("This is number %d \n", foo)
}
```
我们接下来演示的是如何通过反射区打印结构体中的所有对象、打印结构体中的所有字段、调用方法。


``` go
func main() {
	user := User{
		Id:   1,
		Name: "Kiosk",
		Work: Worker{
			Id: 1,
			Occupation: "farmer",
		},
	}

	getType := reflect.TypeOf(user)
	fmt.Println("get type is: ",getType.Name())  // get type is:  User

	getValue := reflect.ValueOf(user)
	fmt.Println("get all Fields is ", getValue) // get all Fields is  {1 Kiosk {1 farmer}}

	// 对字段进行遍历 获取方法的字段
	for i := 0;i < getType.NumField();i++ {
		field := getType.Field(i)
		value := getValue.Field(i).Interface()
		fmt.Printf("%s: %v = %v \n",field.Name,field.Type,value)
	}
	/*
	   	Id: int = 1
		Name: string = Kiosk
		Work: main.Worker = {1 farmer}
	*/

	// 获取方法
	for i := 0;i < getType.NumMethod();i++ {
		m := getType.Method(i)
		fmt.Printf("%s : %v\n",m.Name,m.Type)
	}
	/*
	   ReflectCallFunc : func(main.User)
	   ReflectCallFuncHasArgs : func(main.User, int)
	*/

	// 调用方法
	// 有参数调用
	methodValue := getValue.MethodByName("ReflectCallFuncHasArgs")
	args := []reflect.Value{reflect.ValueOf(2)}
	methodValue.Call(args)

	/*
		This is number 2
	*/

	// 无参数调用
	methodValue = getValue.MethodByName("ReflectCallFunc")
	args = make([]reflect.Value,0)
	methodValue.Call(args)

	/*
		My Id :1 ,My Name :Kiosk ,My Occupation :farmer
	*/
}
```
### 带tag的struct
``` go
type User struct {
	Id      int 		`json:"id"   bson:"id"`
	Name    string		`json:"name" bson:"name"`
}


func main() {
	user := User{
		Id:   1,
		Name: "Kiosk",
	}

	getType := reflect.TypeOf(user)
	fmt.Printf("get type is: %v \n",getType.String())  // get type is:  main.User

	getValue := reflect.ValueOf(user)
	fmt.Println(getValue) // {1 Kiosk}

	name := getValue.FieldByName(getType.Field(1).Name).String()
	fmt.Printf("name is %s \n", name)  // name is Kiosk

	tag := getType.Field(0).Tag
	fmt.Printf("name: %v  tag: '%v'\n", getType.Field(0).Name, tag) // name: Id  tag: 'json:"id"   bson:"id"'
	fmt.Printf("tag is %s, %s \n", tag.Get("json"), tag.Get("bson")) //  tag is id, id 

}
```

## 判断反射值的有效性和空

IsNil()和IsValid() -- 判断反射值的空和有效性

反射值对象（reflect.Value）提供一系列方法进行零值和空判定，如下表所示。

|方 法	| 说 明|
| -- | -- |
| IsNil() bool |	返回值是否为 nil。如果值类型不是通道（channel）、函数、接口、map、指针或 切片时发生 panic，类似于语言层的v== nil操作 |
| IsValid() bool |	判断值是否有效。 当值本身非法时，返回 false，例如 reflect Value不包含任何值，值为 nil 等。|


``` go
func main() {

	//*int的空指针
	var a *int
	fmt.Println("var a *int:", reflect.ValueOf(a).IsNil())  
    // var a *int: true

	//nil值
	fmt.Println("nil:", reflect.ValueOf(nil).IsValid()) 
    // nil: false

	//*int类型的空指针
	fmt.Println("(*int)(nil):", reflect.ValueOf((*int)(nil)).Elem().IsValid()) 
    // (*int)(nil): false

	//实例化一个结构体
	s := struct {}{}

	//尝试从结构体中查找一个不存在的字段
	fmt.Println("不存在的结构体成员:", reflect.ValueOf(s).FieldByName("").IsValid()) 
    // 不存在的结构体成员: false

	//尝试从结构体中查找一个不存在的方法
	fmt.Println("不存在的方法:", reflect.ValueOf(s).MethodByName("").IsValid()) 
    // 不存在的方法: false

	//实例化一个map
	m := map[int]int{}

	//尝试从map中查找一个不存在的键
	fmt.Println("不存在的键:", reflect.ValueOf(m).MapIndex(reflect.ValueOf(3)).IsValid()) 
    // 不存在的键: false

}
```

## 将 chan 反射为 slices
假设一个chan会返回出1、2、3 三个数字，我们可以通过`range`读出chan中的数字。如下

``` golang
func outputInt() chan int {
	a := make(chan int)

	go func() {
		time.Sleep(3*time.Second)
		for i := 0; i < 3; i++{
			a <- i
		}
		close(a)
	}()


	return a
}
```
可以通过反射将chan中的内容反射为切片。

``` golang 
chs := CovertChanToSlice(outputInt())
fmt.Println(chs)
    
// [1,2,3]
```




