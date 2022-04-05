---
title: Go 测试框架 stretchr/testify
author: kiosk
tags:
  - golang
categories:
  - programming
date: 2021-04-29 22:43:00
---
golang的测试框架stretchr/testify 简单的API去检验你的GoLang代码按照你的意愿运行。

项目地址： `https://github.com/stretchr/testify`

<!--more-->

**特性：**
- 易用的断言接口 ([Easy assertions](https://github.com/stretchr/testify#assert-package))
- 接口模拟 ([Mocking](https://github.com/stretchr/testify#mock-package))
- 测试套件接口与性能 ([Testing suite interfaces and functions](https://github.com/stretchr/testify#suite-package))


# 安装
使用 `go get` 安装
``` bash
$ go get github.com/stretchr/testify
```
安装后获得以下包

``` bash
github.com/stretchr/testify/assert
github.com/stretchr/testify/require
github.com/stretchr/testify/mock
github.com/stretchr/testify/suite
github.com/stretchr/testify/http (deprecated
```

# 使用

- assert package
- require package
- mock package
- suite package

**这里面最常用的是前两个 package，他们的唯一差别就是require的函数会直接导致case结束，而assert虽然也标记为case失败，但case不会退出，而是继续往下执行
**
## assert package
`assert` 包提供了一些有用的方法，允许您在Go中编写更好的测试代码。
- 友好输出，更详细的问题描述
- 代码可读性强
- 可以选择用消息注释断言

``` golang
package test

import (
	"github.com/stretchr/testify/assert"
	"testing"
)

func TestCase1(t *testing.T) {
	name := "Bob"
	age := 10

	assert.Equal(t, "Bob", name)
	assert.Equal(t, 20, age,"年龄不相等")
}
```

输出
``` bash
=== RUN   TestCase1
    TestCase1: abc_test.go:18: 
        	Error Trace:	abc_test.go:18
        	Error:      	Not equal: 
        	            	expected: 20
        	            	actual  : 10
        	Test:       	TestCase1
        	Messages:   	年龄不相等
--- FAIL: TestCase1 (0.00s)


Expected :20
Actual   :10
<Click to see difference>


FAIL

Process finished with exit code 1

```


## require package
`require`包提供与assert包相同的全局函数，但它们不会返回布尔结果，而是终止当前测试。


## mock package

`mock`包提供了一种机制，可以方便地编写mock对象，在编写测试代码时可以用它代替真实对象。

如下一个示例测试函数，它测试依赖外部对象`testObj`的一段代码，可以设置期望（证明）并断言它们确实发生了。

例如，一个是消息服务或电子邮件服务，无论何时被调用，都会向客户端发送电子邮件。如果我们正在积极地开发我们的代码库，可能每天会运行数百次测试，但我们不希望每天向客户发送数百封电子邮件或消息，因为那样他们可能会不高兴。

> 那么，我们要如何使用 testify 包来模拟呢？

让我们来看一下如何将 `mocks` 应用到一个相当简单的例子中。在这个例子中，我们有一个系统会尝试向客户收取产品或服务的费用。当 `ChargeCustomer()` 被调用时，它将随后调用 `Message Service`，向客户发送 SMS 文本消息来通知他们已经被收取的金额。

``` golang
// MessageService 通知客户被收取的费用
type MessageService interface {
	SendChargeNotification(int) error
}

// SMSService 是 MessageService 的实现
type SMSService struct{}

// MyService 使用 MessageService 来通知客户
type MyService struct {
	messageService MessageService
}

// SendChargeNotification 通过 SMS 来告知客户他们被收取费用
// 这就是我们将要模拟的方法
func (sms SMSService) SendChargeNotification(value int) error {
	fmt.Println("Sending Production Charge Notification")
	return nil
}

// ChargeCustomer 向客户收取费用
// 在真实系统中，我们会模拟这个
// 但是在这里，我想在每次运行时都赚点钱
func (a MyService) ChargeCustomer(value int) error {
	a.messageService.SendChargeNotification(value)
	fmt.Printf("Charging Customer For the value of %d\n", value)
	return nil
}

// 一个 "Production" 例子
func main() {
	fmt.Println("Hello World")

	smsService := SMSService{}
	myService := MyService{smsService}
	myService.ChargeCustomer(100)
}
```

那么，我们如何进行测试以确保我们不会让客户疯掉？好吧，我们通过创建一个新的 struct 称之为 `smsServiceMock` ，用来模拟我们的 `SMSService`，并且将 `mock.Mock` 添加到它的字段列表中。

然后我们将改写 `SendChargeNotification` 方法，这样它就不会向我们的客户发送通知并返回 nil 错误。

最后，我们创建 `TestChargeCustomer` 测试函数，接着实例化一个新的类型实例 `smsServiceMock` 并指定 `SendChargeNotification `在被调用时应该做什么。

``` golang
// smsServiceMock
type smsServiceMock struct {
	mock.Mock
}

// 我们模拟的 smsService 方法
func (m smsServiceMock) SendChargeNotification(value int) error {
	fmt.Println("Mocked charge notification function")
	fmt.Printf("Value passed in: %d\n", value)
	// 这将记录方法被调用以及被调用时传进来的参数值
	args := m.Called(value)
	// 它将返回任何我需要返回的
	// 这种情况下模拟一个 SMS Service Notification 被发送出去
	args.Bool(0)
	return nil
}

// TestChargeCustomer 是个奇迹发生的地方
// 在这里我们将创建 SMSService mock
func TestChargeCustomer(t *testing.T) {
	var smsService smsServiceMock

	// 然后我们将定义当 100 传递给 SendChargeNotification 时，需要返回什么
	// 在这里，我们希望它在成功发送通知后返回 true
	smsService.On("SendChargeNotification", 100).Return(true)

	// 接下来，我们要定义要测试的服务
	myService := MyService{smsService}
	// 然后调用方法
	myService.ChargeCustomer(100)

	// 最后，我们验证 myService.ChargeCustomer 调用了我们模拟的 SendChargeNotification 方法
	smsService.AssertExpectations(t)
}
```

所以，当我们运行 `go test ./... -v `时，我们应该看到以下输出：

``` golang
go test ./... -v
=== RUN   TestChargeCustomer
Mocked charge notification function
Value passed in: 100
Charging Customer For the value of 100
--- PASS: TestChargeCustomer (0.00s)
    main_test.go:33: PASS:      SendChargeNotification(int)
PASS
ok      _/Users/elliot/Documents/Projects/tutorials/golang/go-testify-tutorial  0.012s
```
这证明我们的 myService.ChargeCustomer() 方法按照我们所期望的方式在运行！

我们现在已经能够使用模拟的方法来全面测试更复杂的项目。值得注意的是，此技术可用于各种不同的系统，例如模拟数据库查询或者是如何与其他 API 交互。总的来说，模拟是非常强大的手段

## suite package 

这个包不太常用
