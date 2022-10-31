# 设计模式（Design Pattern）

设计模式（Design pattern）代表了最佳的实践，通常被有经验的面向对象的软件开发人员所采用。设计模式是软件开发人员在软件开发过程中面临的一般问题的解决方案。这些解决方案是众多软件开发人员经过相当长的一段时间的试验和错误总结出来的。

<!--more-->

# 设计模式概述

"设计模式"总的来说有设计原则6个.

- **单一职责原则(Single Responsibility Principle, SRP)** : 每个模块或类都应该对软件提供的功能的一部分负责，而这个责任应该完全由类来封装。它的所有服务都应严格遵守这一职责。
- **开闭原则(Open Close Principle, OCP)** : 软件中的对象(类、模块、函数等)对扩展是开放的，对修改是封闭的。
- **里氏替换原则(Liskov Substitution Principle, LSP)** : 所有使用基类的地方必须能透明地使用其子类的对象
- **依赖倒转原则(Dependence Inversion Principle, DIP)** : 是指一种特定的解耦（传统的依赖关系建立在高层次上，而具体的策略设置则应用在低层次的模块上）形式，使得高层次的模块不依赖于低层次的模块的实现细节，依赖关系被颠倒（反转），从而使得低层次模块依赖于高层次模块的需求抽象。
- **接口隔离原则(Interface Segregation Principle, ISP)** : 客户端不应该依赖它不需要的接口。
- **迪米特法则(Law of Demeter, LoD), 最少知识原则(Principle of Least Knowledge)** : 1. 每个对象应该对其他对象尽可能最少的知道 2. 每个对象应该仅和其朋友通信；不和陌生人通信 3. 仅仅和直接朋友通信

--------------------


# 创建型
## 单例模式 (Singleton)


- <font color='#200000'>定义以及使用场景</font>

1. 确保某一个类只有一个实例，而且向整个系统提供这个实例
2. 确保某个类有且仅有一个对象的场景，避免产生多个对象消耗过多的资源；或者某种类型的对象应该有且只有一个。（如 Logger 实例、Config 实例等）

- <font color='#200000'>实现单例模式的几个关键点</font>

1. 构造函数不对外开放，一般为private
2. 通过一个静态方法或者枚举返回单例类对象
3. 确保单例类的对象有且只有一个，尤其是在多线程环境下
4. 确保单例类对象在反序列化时不会重新构建对象


- <font color='#200000'>饥汉模式</font>

直接创建好对象，这样不需要判断为空，同时也是线程安全。唯一的缺点是在导入包的同时会创建该对象，并持续占有在内存中。

``` golang
var instance Tool

func GetInstance() *Tool {
    return &instance
}
```

- <font color='#200000'> 懒汉模式 (Lazy Loading) </font>

只有需要时才会初始化，在一定程度上节约了资源。如果不加锁的话非线程安全，即在多线程下可能会创建多次对象。**懒汉方式是开源项目中使用最多的**

``` golang
// private
var instance *singleton
var mu sync.Mutex

// public
func GetInstance() *singleton {
	mu.Lock()
	defer mu.Unlock()

	if instance == nil {
		instance = &singleton{}
	}
	return instance
}
```

- <font color='#200000'> DCL(双重检查)模式 (推荐) </font>

DCL的优点就是资源利用率高，只有第一次执行getInstance才会初始化。


``` golang
var lock sync.Mutex
var instance Tool

/**
* 第一次判断不加锁，第二次加锁保证线程安全，一旦对象建立后,获取对象就不用加锁了
*/
func GetInstance() *Tool {
    if instance == nil {
        lock.Lock()

        if instance == nil {
            instance = new(Tool)
        }

        lock.Unlock()
    }

    return instance
}
```

或者在golang中还可以使用 `sync.Once` 保证单例。

``` golang
var once sync.Once

func GetInstance() *Tool {
    once.Do(func() {
        instance = new(Tool)
	})
    return instance
}
``` 




## 工厂方法模式(Factory method)

- <font color='#808080'> 定义以及使用场景</font>

创建一个用户创建对象的接口，让子类决定实例化哪个类。工厂方法使一个类的实例化延迟到其子类。

**使用场景**：

1. 工厂方法模式通过依赖抽象来达到解耦的效果，并且将实例化的任务交给子类去完成，有非常好的扩展性
2. 在任何需要生成复杂对象的地方，都可以使用工厂方法模式。复杂对象适合使用工厂模式，用new就可以完成创建的对象无需使用工厂方法
3. 工厂方法模式的应用非常广泛，然而缺点也很明显，就是每次我们为工厂方法添加新的产品时，都需要编写一个新的产品类，所以要根据实际情况来权衡是否要用工厂方法模式

类似我要造汽车，将造汽车的通用的几个方法定义好，就可以创建一个接口。任何实现了这套造汽车标准的厂商都可以被初始化。并造出一辆汽车。

- <font color='#808080'> 举例实现工厂方法模式 </font>

假设我们在做一款小型翻译软件，软件可以将德语、英语、日语都翻译成目标中文，并显示在前端。

- <font color='#200000'> 简单工厂模式 </font>

我们会有三个具体的语言翻译结构体，或许以后还有更多，但现在分别是GermanTranslater、EnglishTranslater、JapaneseTranslater，他们都共同实现了一个接口Translator。

``` go
//翻译接口
type Translator interface {
	Translate(string) string
}

//德语翻译类
type GermanTranslator struct{}
func (*GermanTranslator) Translate(words string) string {
	return "德语"
}

//英语翻译类
type EnglishTranslator struct{}
func (*EnglishTranslator) Translate(words string) string {
	return "英语"
}
```

接下来在程序入口获取用户输入的文本，并将其翻译


``` go
func main() {
	defer func() {
		if err := recover(); err != nil {
			fmt.Println(err)
		}
		time.Sleep(3 * time.Second)
	}()

	var lan int
	fmt.Printf("%s ,%s", "以下是可翻译的语言种类，请输入代表数字", "1：德语、2：英语")
	_, _ = fmt.Scanln(&lan)

	fmt.Println("请输入要翻译成中文的文本：")
	var inputWords string
	_, _ = fmt.Scanln(&inputWords)

	var translator Translator

	//根据不同的语言种类，实例化不同的翻译类
	switch lan {
	case 1:
		translator = new(GermanTranslator)
	case 2:
		translator = new(EnglishTranslator)
	default:
		panic("no such translator")
	}

	fmt.Println(translator.Translate(inputWords))
}

```

**缺点**

1. 违背了开闭原则，以后还可能有法语、俄语、阿拉伯语等其他翻译器，每一次添加翻译器都要在客户端代码增加对应的switch分支，维护成本高。倘若还有不止一处调用了创建逻辑，还要维护多处代码。
2. 违背了单一职责原则，客户端处理类的职责应该只是负责接收用户的输入并将其打印，现在还负责翻译类的创建逻辑，导致这个类的职责过多。

改造

``` go
// 工厂函数
func CreateTranslator(lan int) Translator {
	var translator Translator
	
	switch lan {
	case 1:
		translator = new(GermanTranslator)
	case 2:
		translator = new(EnglishTranslator)
	default:
		panic("no such translator")
	}

	return translator
}

// 主函数
...
    fmt.Println("请输入要翻译成中文的文本：")
	var inputWords string
	fmt.Scanln(&inputWords)

    //客户端只关注如何获取翻译类，而不用关注创建翻译类的细节
	translator:=CreateTranslator(lan)

	fmt.Println(translator.Translate(inputWords))
...
```

--------------------


## 建造者模式(Builder)


- **定义以及使用场景** 

将一个复杂对象的构建与它的表示分离，使得同样的构建过程可以创建不同的表示。

**使用场景**

1. 相同的方法，不同的执行顺序，产生不同的事件结果时
2. 多个部件或零件，都可以装配到一个对象中，但是产生的运行结果却又不相同时
产品类非常复杂，或者产品类中的调用顺序不同产生了不同的结果，这个时候使用建造者模式非常合适
3. 当初始化一个对象特别复杂，如参数多，且很多参数都具有默认值时

> **Product** 产品类——产品的抽象类
> **Builder** 抽象Builder类，规范产品的组建，一般由子类实现具体的组建过程
> **ConcreteBuilder** 具体的Builder类
> **Director** 统一组装过程

- **举个例子** 

我们需要创建汽车，而汽车有轮胎的个数以及车身的颜色可定制，那么用Builder模式可以这样。

我们的目标是建一辆车

``` go 
// 定义一辆车
type Car struct {
	Wheels 	string
	Chassis string
}

func (c *Car) Show() {
	fmt.Println("Builder Complete ...")
	fmt.Printf("Wheels : %s Chassis: %s \n", c.Wheels, c.Chassis)
}
```

设计出完整的建设规划

``` go
// 为建造者实现 Builder 接口
type Builder interface {
	NewProduct()       	// 创建一个空产品
	BuildWheels()      	// 建造轮子
	BuildChassis()    	// 建造底盘
	GetResult() interface{}  // 获取建造好的产品
}
```

按照Builder规划一个大型项目构造者CarBuilder, 包含如何具体实现Build

``` go
// 定义汽车建造项目 CarBuilder
type CarBuilder struct {
	Car *Car
}

func (cb *CarBuilder) GetResult() interface{} {
	return cb.Car
}

func (cb *CarBuilder) NewProduct() {
	cb.Car = new(Car)
}

func (cb *CarBuilder) BuildWheels() {
	cb.Car.Wheels = "米其林轮胎"
}

func (cb *CarBuilder) BuildChassis() {
	cb.Car.Chassis = "沃尔沃底盘"
}
```

下面要把具体建造者传入指挥者:

``` go
// 把建造者传入指挥者
type Director struct {
	builder Builder
}

func (d *Director) SetBuilder(builder Builder) {
	d.builder = builder
}

```

建造实施

``` go
func (d *Director) CarBuilderImpl() *Car {
	d.builder.NewProduct()
	d.builder.BuildChassis()
	d.builder.BuildWheels()
	return d.builder.GetResult().(*Car)
}
```

完整过程

``` go
func main() {
	// 创建一个指挥者
	director := new(Director)
	// 创建建造者
	builder := new(CarBuilder)

	director.SetBuilder(builder)
	car := director.CarBuilderImpl()
	car.Show()
}
```

--------------------


## 原型模式(Prototype)


- <font color='#808080'>定义以及使用场景</font>

原型模式用于创建重复的对象。当一个类在创建时开销比较大时(比如大量数据准备，数据库连接)，我们可以缓存该对象，当下一次调用时，返回该对象的克隆。

> **不过大多数原型模式不在日常中使用，一般会使用 `sync.pool` 替代，详见 [Worker Pool in Golang](https://kiosk007.top/2021/03/21/Worker-Pool-in-Golang/)**

**使用场景**

1. 类初始化需要消耗非常多的资源，这个资源包括数据、硬件资源等，通过原型拷贝避免这些消耗
2. 原型模式是在内存中二进制流的拷贝，要比直接new一个对象性能好很多，特别是要在一个循环体内产生大量的对象时，原型模式可以更好滴体现其优点
3. 一个对象需要提供给其他对象访问，而且各个调用者可能需要修改其值时，可以考虑使用原型模式拷贝多个对象供调用者使用，即保护性拷贝


- <font color='#200000'>举个例子</font>

定义一个原型管理器

``` go
// 样品（原型）Clone能力约定类
type Cloneable interface {
	Clone() Cloneable
}

// 样品（原型）管理器类
type PrototypeManager struct {
	prototypes map[string]Cloneable
}

func (that *PrototypeManager) Set( cloneName string, cloneable Cloneable) {
	that.prototypes[cloneName] = cloneable
}

func (that *PrototypeManager) Get(cloneName string) (prototype Cloneable, err error) {
	if prototype, ok:=that.prototypes[cloneName]; ok {
		return prototype, nil
	} else {
		return nil, errors.New(fmt.Sprintf("%s 不存在", cloneName))
	}
}
```

获取一个样品（原型）管理器

``` go
// 获取一个样品（原型）管理器
func NewPrototypeManager () *PrototypeManager {
	return &PrototypeManager{
		prototypes:make(map[string]Cloneable),
	}
}

var manager *PrototypeManager
```

定义一个样品原型

``` go
// 定义一个样品（原型） 实现 Clone 方法 相当于把自己做成了一个样品
type PT struct {}

func(that *PT) Clone() Cloneable {
	temp := *that
	return &temp
}

```

测试


``` go
func Test() {
	prototypeOne,_ := manager.Get("prototypeOne")
	prototypeTwo := prototypeOne.Clone()
	prototypeThree := prototypeOne.Clone()
	fmt.Printf(" prototypeOne地址:%v \n " +
		"prototypeTwo地址: %v \n " +
		"prototypeThree地址: %v \n", &prototypeOne, &prototypeTwo, &prototypeThree)
}

func main() {
	manager = NewPrototypeManager()
	pt1 := &PT{}
	manager.Set("prototypeOne", pt1)
	Test()
}

// 输出
 prototypeOne地址:   0xc000010200 
 prototypeTwo地址:   0xc000010210 
 prototypeThree地址: 0xc000010220 

```

----

# 结构型

## 代理模式 (Proxy)

- <font color='#808080'> 定义以及使用场景 </font>

在软件设计中，使用代理模式的例子也很多，例如，要访问的远程对象比较大（如视频或大图像等），其下载要花很多时间。还有因为安全原因需要屏蔽客户端直接访问真实对象，如某单位的内部数据库等。

**使用场景**

由于某些原因需要给某对象提供一个代理以控制对该对象的访问。这时，访问对象不适合或者不能直接引用目标对象，代理对象作为访问对象和目标对象之间的中介。

``` go

// IUser IUser
type IUser interface {
	Login(username, password string) error
}

// User 用户
type User struct {
}

// Login 用户登录
func (u *User) Login(username, password string) error {
	// 不实现细节
	return nil
}

// UserProxy 代理类
type UserProxy struct {
	user *User
}

// NewUserProxy NewUserProxy
func NewUserProxy(user *User) *UserProxy {
	return &UserProxy{
		user: user,
	}
}

// Login 登录，和 user 实现相同的接口
func (p *UserProxy) Login(username, password string) error {
	// before 这里可能会有一些统计的逻辑
	start := time.Now()

	// 这里是原有的业务逻辑
	if err := p.user.Login(username, password); err != nil {
		return err
	}

	// after 这里可能也有一些监控统计的逻辑
	log.Printf("user login cost time: %s", time.Now().Sub(start))

	return nil
}
```

- 测试调用


``` go
func Test_UserLogin() {
	proxy := NewUserProxy(&User{})

	if err := proxy.Login("test", "password"); err != nil {
		panic(err)
	}
}
```


---------------------



## 桥接模式 (Bridge)

- <font color='#808080'> 定义以及使用场景 </font> 

桥接（Bridge）模式的定义如下：将抽象与实现分离，使它们可以独立变化。它是用组合关系代替继承关系来实现，从而降低了抽象和实现这两个可变维度的耦合度。

**使用场景**

1. 抽象化（Abstraction）角色：定义抽象类，并包含一个对实现化对象的引用。
2. 扩展抽象化（Refined Abstraction）角色：是抽象化角色的子类，实现父类中的业务方法，并通过组合关系调用实现化角色中的业务方法。
3. 实现化（Implementor）角色：定义实现化角色的接口，供扩展抽象化角色调用。
4. 具体实现化（Concrete Implementor）角色：给出实现化角色接口的具体实现。

``` go
package bridge

// IMsgSender IMsgSender
type IMsgSender interface {
	Send(msg string) error
}

// EmailMsgSender 发送邮件
// 可能还有 电话、短信等各种实现
type EmailMsgSender struct {
	emails []string
}

// NewEmailMsgSender NewEmailMsgSender
func NewEmailMsgSender(emails []string) *EmailMsgSender {
	return &EmailMsgSender{emails: emails}
}

// Send Send
func (s *EmailMsgSender) Send(msg string) error {
	// 这里去发送消息
	return nil
}

// INotification 通知接口
type INotification interface {
	Notify(msg string) error
}

// ErrorNotification 错误通知
// 后面可能还有 warning 各种级别
type ErrorNotification struct {
	sender IMsgSender
}

// NewErrorNotification NewErrorNotification
func NewErrorNotification(sender IMsgSender) *ErrorNotification {
	return &ErrorNotification{sender: sender}
}

// Notify 发送通知
func (n *ErrorNotification) Notify(msg string) error {
	return n.sender.Send(msg)
}
```

单元测试
``` go
func TestErrorNotification_Notify(t *testing.T) {
	sender := NewEmailMsgSender([]string{"test@test.com"})
	n := NewErrorNotification(sender)
	err := n.Notify("test msg")

	assert.Nil(t, err)
}
```


--------------------


## 装饰器模式 (Decorator)

- <font color='#808080'>定义以及使用场景</font>
装饰器（Decorator）模式的定义：指在不改变现有对象结构的情况下，动态地给该对象增加一些职责（即增加其额外功能）的模式，它属于对象结构型模式。

- **使用场景**
在软件开发过程中，有时想用一些现存的组件。这些组件可能只是完成了一些核心功能。但在不改变其结构的情况下，可以动态地扩展其功能。所有这些都可以釆用装饰器模式来实现。

``` go
// IDraw IDraw
type IDraw interface {
	Draw() string
}

// Square 正方形
type Square struct{}

// Draw Draw
func (s Square) Draw() string {
	return "this is a square"
}

// ColorSquare 有颜色的正方形
type ColorSquare struct {
	square IDraw
	color  string
}

// NewColorSquare NewColorSquare
func NewColorSquare(square IDraw, color string) ColorSquare {
	return ColorSquare{color: color, square: square}
}

// Draw Draw
func (c ColorSquare) Draw() string {
	return c.square.Draw() + ", color is " + c.color
}

```

单元测试

``` golang
func Test_Decorator() {
	sq := Square{}
	csq := NewColorSquare(sq, "red")
	got := csq.Draw()
	assert.Equal(t, "this is a square, color is red", got)
}
```



## 适配器模式 (Adapter)

- <font color='#808080'>定义以及使用场景</font>
适配器模式（Adapter）的定义如下：将一个类的接口转换成客户希望的另外一个接口，使得原本由于接口不兼容而不能一起工作的那些类能一起工作。适配器模式分为类结构型模式和对象结构型模式两种，前者类之间的耦合度比后者高，且要求程序员了解现有组件库中的相关组件的内部结构，所以应用相对较少些。

- **使用场景**

需要开发的具有某种业务功能的组件在现有的组件库中已经存在，但它们与当前系统的接口规范不兼容，如果重新开发这些组件成本又很高，这时用适配器模式能很好地解决这些问题。


``` golang
// ICreateServer 创建云主机
type ICreateServer interface {
	CreateServer(cpu, mem float64) error
}

// AWSClient aws sdk
type AWSClient struct{}

// RunInstance 启动实例
func (c *AWSClient) RunInstance(cpu, mem float64) error {
	fmt.Printf("aws client run success, cpu： %f, mem: %f", cpu, mem)
	return nil
}

// AwsClientAdapter 适配器
type AwsClientAdapter struct {
	Client AWSClient
}

// CreateServer 启动实例
func (a *AwsClientAdapter) CreateServer(cpu, mem float64) error {
	a.Client.RunInstance(cpu, mem)
	return nil
}

// AliyunClient aliyun sdk
type AliyunClient struct{}

// CreateServer 启动实例
func (c *AliyunClient) CreateServer(cpu, mem int) error {
	fmt.Printf("aliyun client run success, cpu： %d, mem: %d", cpu, mem)
	return nil
}

// AliyunClientAdapter 适配器
type AliyunClientAdapter struct {
	Client AliyunClient
}

// CreateServer 启动实例
func (a *AliyunClientAdapter) CreateServer(cpu, mem float64) error {
	a.Client.CreateServer(int(cpu), int(mem))
	return nil
}

```

单元测试

``` golang
func Test_Adapter(t *testing.T) {
	// 确保 adapter 实现了目标接口
	var a ICreateServer = &AliyunClientAdapter{
		Client: AliyunClient{},
	}

	a.CreateServer(1.0, 2.0)
}
```




## 门面模式 (Facade)

- <font color='#808080'>定义以及使用场景</font>

外观（Facade）模式又叫作门面模式，是一种通过为多个复杂的子系统提供一个一致的接口，而使这些子系统更加容易被访问的模式。该模式对外有一个统一接口，外部应用程序不用关心内部子系统的具体细节，这样会大大降低应用程序的复杂度，提高了程序的可维护性。

- **使用场景**

我们都在有意无意的大量使用外观模式。只要是高层模块需要调度多个子系统（2个以上的类对象），我们都会自觉地创建一个新的类封装这些子系统，提供精简的接口，让高层模块可以更加容易地间接调用这些子系统的功能。尤其是现阶段各种第三方SDK、开源类库，很大概率都会使用外观模式。


``` golang
// IUser 用户接口
type IUser interface {
	Login(phone int, code int) (*User, error)
	Register(phone int, code int) (*User, error)
}

// IUserFacade 门面模式
type IUserFacade interface {
	LoginOrRegister(phone int, code int) error
}

// User 用户
type User struct {
	Name string
}

// UserService UserService
type UserService struct {}

// Login 登录
func (u UserService) Login(phone int, code int) (*User, error) {
	// 校验操作 ...
	return &User{Name: "test login"}, nil
}

// Register 注册
func (u UserService) Register(phone int, code int) (*User, error) {
	// 校验操作 ...
	// 创建用户
	return &User{Name: "test register"}, nil
}

// LoginOrRegister 登录或注册
func (u UserService)LoginOrRegister(phone int, code int) (*User, error) {
	user, err := u.Login(phone, code)
	if err != nil {
		return nil, err
	}

	if user != nil {
		return user, nil
	}

	return u.Register(phone, code)
}
```

单元测试


``` golang 
func TestUserService_LoginOrRegister(t *testing.T) {
	service := UserService{}
	user, err := service.LoginOrRegister(13001010101, 1234)
	assert.NoError(t, err)
	assert.Equal(t, &User{Name: "test login"}, user)
}

```




## 享元模式


- <font color='#808080'>定义以及使用场景</font>

享元（Flyweight）模式的定义：运用共享技术来有效地支持大量细粒度对象的复用。它通过共享已经存在的对象来大幅度减少需要创建的对象数量、避免大量相似类的开销，从而提高系统资源的利用率。

享元模式的主要优点是：相同对象只要保存一份，这降低了系统中对象的数量，从而降低了系统中细粒度对象给内存带来的压力。

- **使用场景**

在面向对象程序设计过程中，有时会面临要创建大量相同或相似对象实例的问题。创建那么多的对象将会耗费很多的系统资源，它是系统性能提高的一个瓶颈。

例如，围棋和五子棋中的黑白棋子，图像中的坐标点或颜色，局域网中的路由器、交换机和集线器，教室里的桌子和凳子等。这些对象有很多相似的地方，如果能把它们相同的部分提取出来共享，则能节省大量的系统资源，这就是享元模式的产生背景。


``` golang

var units = map[int]*ChessPieceUnit{
	1: {
		ID:    1,
		Name:  "車",
		Color: "red",
	},
	2: {
		ID:    2,
		Name:  "炮",
		Color: "red",
	},
	// ... 其他棋子
}

// ChessPieceUnit 棋子享元
type ChessPieceUnit struct {
	ID    uint
	Name  string
	Color string
}

// NewChessPieceUnit 工厂
func NewChessPieceUnit(id int) *ChessPieceUnit {
	return units[id]
}

// ChessPiece 棋子
type ChessPiece struct {
	Unit *ChessPieceUnit
	X    int
	Y    int
}

// ChessBoard 棋局
type ChessBoard struct {
	chessPieces map[int]*ChessPiece
}

// NewChessBoard 初始化棋盘
func NewChessBoard() *ChessBoard {
	board := &ChessBoard{chessPieces: map[int]*ChessPiece{}}
	for id := range units {
		board.chessPieces[id] = &ChessPiece{
			Unit: NewChessPieceUnit(id),
			X:    0,
			Y:    0,
		}
	}
	return board
}

// Move 移动棋子
func (c *ChessBoard) Move(id, x, y int) {
	c.chessPieces[id].X = x
	c.chessPieces[id].Y = y
}
```

单元测试

``` golang 
func TestNewChessBoard(t *testing.T) {
	board1 := NewChessBoard()
	board1.Move(1, 1, 2)
	board2 := NewChessBoard()
	board2.Move(2, 2, 3)

	assert.Equal(t, board1.chessPieces[1].Unit, board2.chessPieces[1].Unit)
	assert.Equal(t, board1.chessPieces[2].Unit, board2.chessPieces[2].Unit)
}
```




# 行为型

## 观察者模式 (Observe)

- <font color='#808080'> 定义以及使用场景 </font>
观察者（Observer）模式的定义：指多个对象间存在一对多的依赖关系，当一个对象的状态发生改变时，所有依赖于它的对象都得到通知并被自动更新。这种模式有时又称作发布-订阅模式、模型-视图模式，它是对象行为型模式。

- **使用场景**

例如，Excel 中的数据与折线图、饼状图、柱状图之间的关系；MVC 模式中的模型与视图的关系；事件模型中的事件源与事件处理者。所有这些，如果用观察者模式来实现就非常方便。

``` go
// ISubject subject
type ISubject interface {
	Register(observer IObserver)
	Remove(observer IObserver)
	Notify(msg string)
}

// IObserver 观察者
type IObserver interface {
	Update(msg string)
}

// Subject Subject
type Subject struct {
	observers []IObserver
}

// Register 注册
func (sub *Subject) Register(observer IObserver) {
	sub.observers = append(sub.observers, observer)
}

// Remove 移除观察者
func (sub *Subject) Remove(observer IObserver) {
	for i, ob := range sub.observers {
		if ob == observer {
			sub.observers = append(sub.observers[:i], sub.observers[i+1:]...)
		}
	}
}

// Notify 通知
func (sub *Subject) Notify(msg string) {
	for _, o := range sub.observers {
		o.Update(msg)
	}
}

// Observer1 Observer1
type Observer1 struct{}

// Update 实现观察者接口
func (Observer1) Update(msg string) {
	fmt.Printf("Observer1: %s\n", msg)
}

// Observer2 Observer2
type Observer2 struct{}

// Update 实现观察者接口
func (Observer2) Update(msg string) {
	fmt.Printf("Observer2: %s\n", msg)
}
```

单元测试

``` golang
func TestNewChessBoard(t *testing.T) {
	sub := &Subject{}
	sub.Register(&Observer1{})
	sub.Register(&Observer2{})
	sub.Notify("hi")
}
```

## 模板方法模式 (Template Method)

- <font color='#808080'> 定义以及使用场景 </font>

模板方法（Template Method）模式的定义如下：定义一个操作中的算法骨架，而将算法的一些步骤延迟到子类中，使得子类可以不改变该算法结构的情况下重定义该算法的某些特定步骤。它是一种类行为型模式。

**使用场景**

例如，一个人每天会起床、吃饭、做事、睡觉等，其中“做事”的内容每天可能不同。我们把这些规定了流程或格式的实例定义成模板，允许使用者根据自己的需求去更新它，例如，简历模板、论文模板、Word 中模板文件等。


**举个 🌰**
假设我现在要做一个短信推送的系统，那么需要

1. 检查短信字数是否超过限制
2. 检查手机号是否正确
3. 发送短信
4. 返回状态
我们可以发现，在发送短信的时候由于不同的供应商调用的接口不同，所以会有一些实现上的差异，但是他的算法（业务逻辑）是固定的

``` golang
// ISMS ISMS
type ISMS interface {
	send(content string, phone int) error
}

// SMS 短信发送基类
type sms struct {
	ISMS
}

// Valid 校验短信字数
func (s *sms) Valid(content string) error {
	if len(content) > 63 {
		return fmt.Errorf("content is too long")
	}
	return nil
}

// Send 发送短信
func (s *sms) Send(content string, phone int) error {
	if err := s.Valid(content); err != nil {
		return err
	}

	// 调用子类的方法发送短信
	return s.send(content, phone)
}

// TelecomSms 走电信通道
type TelecomSms struct {
	*sms
}

// NewTelecomSms NewTelecomSms
func NewTelecomSms() *TelecomSms {
	tel := &TelecomSms{}
	// 这里有点绕，是因为 go 没有继承，用嵌套结构体的方法进行模拟
	// 这里将子类作为接口嵌入父类，就可以让父类的模板方法 Send 调用到子类的函数
	// 实际使用中，我们并不会这么写，都是采用组合+接口的方式完成类似的功能
	tel.sms = &sms{ISMS: tel}
	return tel
}

func (tel *TelecomSms) send(content string, phone int) error {
	fmt.Println("send by telecom success")
	return nil
}
```

单元测试

``` golang
func Test_sms_Send(t *testing.T) {
	tel := NewTelecomSms()
	err := tel.Send("test", 1239999)
	assert.NoError(t, err)
}
``` 


## 命令模式 (Command)

- <font color='#808080'> 定义以及使用场景 </font>

它可将请求或简单操作转换为一个对象。此类转换让你能够延迟进行或远程执行请求， 还可将其放入队列中。

**使用场景**

1. 需要抽象出待执行的行动，然后以参数的形式提供出来——类似于过程设计中的回调机制，而命令模式正是回调机制的一个面向对象的代替品。
2. 在不同的时刻指定、排列和执行请求，一个命令对象可以有与初始请求无关的生存期
3. 需要支持事务操作


- <font color='#200000'> 举个例子 </font>

**电视遥控器**:

遥控器从实现 ON 命令对象并以电视机作为接收者入手。 当在此命令上调用 execute执行方法时， 方法会调用 TV.on打开电视函数。 最后的工作是定义请求者： 这里实际上有两个请求者： 遥控器和电视机。 两者都将嵌入 ON 命令对象。


- 实现 command 和 device 

``` go
// command define
type command interface {
	execute()
}
type button struct {
	command command
}
func (b *button) press() {
	b.command.execute()
}

// device define ， 设备可执行的命令
type device interface {
	on()
	off()
}

// 具体实现要执行命令
type onCommand struct {
	device device
}
func (c *onCommand) execute() {
	c.device.on()
}


type offCommand struct {
	device device
}
func (c *offCommand) execute() {
	c.device.off()
}

```

创建命令的发出者和执行者

``` go
// 命令的执行者
type tv struct {
	isRunning bool
}

func (t *tv) on() {
	t.isRunning = true
	fmt.Println("Turning tv on")
}

func (t *tv) off() {
	t.isRunning = false
	fmt.Println("Turning tv off")
}

// 执行者就是按下按钮
func main() {
	tv := &tv{}

	onButton := &button{
		command: &onCommand{ device: tv },
	}
	onButton.press()

}

```

## 策略模式 (Strategy)

- <font color='#808080'>定义以及使用场景</font>

策略模式（Strategy Pattern）定义一组算法，将每个算法都封装起来，并且使它们之间可以互换。

**使用场景**

在项目开发中，我们经常要根据不同的场景，采取不同的措施，也就是不同的策略。比如，假设我们需要对 a、b 这两个整数进行计算，根据条件的不同，需要执行不同的计算方式。我们可以把所有的操作都封装在同一个函数中，然后通过 if ... else ... 的形式来调用不同的计算方式，这种方式称之为硬编码。

在实际应用中，随着功能和体验的不断增长，我们需要经常添加 / 修改策略，这样就需要不断修改已有代码，不仅会让这个函数越来越难维护，还可能因为修改带来一些 bug。所以为了解耦，需要使用策略模式，定义一些独立的类来封装不同的算法，每一个类封装一个具体的算法（即策略）。

``` golang

package strategy

// 策略模式

// 定义一个策略类
type IStrategy interface {
  do(int, int) int
}

// 策略实现：加
type add struct{}

func (*add) do(a, b int) int {
  return a + b
}

// 策略实现：减
type reduce struct{}

func (*reduce) do(a, b int) int {
  return a - b
}

// 具体策略的执行者
type Operator struct {
  strategy IStrategy
}

// 设置策略
func (operator *Operator) setStrategy(strategy IStrategy) {
  operator.strategy = strategy
}

// 调用策略中的方法
func (operator *Operator) calculate(a, b int) int {
  return operator.strategy.do(a, b)
}
```
在上述代码中，我们定义了策略接口 IStrategy，还定义了 add 和 reduce 两种策略。最后定义了一个策略执行者，可以设置不同的策略，并执行，例如：

``` golang

func TestStrategy(t *testing.T) {
  operator := Operator{}

  operator.setStrategy(&add{})
  result := operator.calculate(1, 2)
  fmt.Println("add:", result)

  operator.setStrategy(&reduce{})
  result = operator.calculate(2, 1)
  fmt.Println("reduce:", result)
}
```

## 职责链条模式（Chain of Responsibility）

- <font color='#808080'>定义以及使用场景</font>

为了避免请求发送者与多个请求处理者耦合在一起，于是将所有请求的处理者通过前一对象记住其下一个对象的引用而连成一条链；当有请求发生时，可将请求沿着这条链传递，直到有对象处理它为止。


**使用场景**

如总线网中数据报传送，每台计算机根据目标地址是否同自己的地址相同来决定是否接收；还有异常处理中，处理程序根据异常的类型决定自己是否处理该异常；还有 Struts2 的拦截器、JSP 和 Servlet 的 Filter 等，所有这些，都可以考虑使用责任链模式来实现。



``` golang
// SensitiveWordFilter 敏感词过滤器，判定是否是敏感词
type SensitiveWordFilter interface {
	Filter(content string) bool
}

// SensitiveWordFilterChain 职责链
type SensitiveWordFilterChain struct {
	filters []SensitiveWordFilter
}

// AddFilter 添加一个过滤器
func (c *SensitiveWordFilterChain) AddFilter(filter SensitiveWordFilter) {
	c.filters = append(c.filters, filter)
}

// Filter 执行过滤
func (c *SensitiveWordFilterChain) Filter(content string) bool {
	for _, filter := range c.filters {
		// 如果发现敏感直接返回结果
		if filter.Filter(content) {
			return true
		}
	}
	return false
}

// AdSensitiveWordFilter 广告
type AdSensitiveWordFilter struct{}

// Filter 实现过滤算法
func (f *AdSensitiveWordFilter) Filter(content string) bool {
	// TODO: 实现算法
	return false
}

// PoliticalWordFilter 政治敏感
type PoliticalWordFilter struct{}

// Filter 实现过滤算法
func (f *PoliticalWordFilter) Filter(content string) bool {
	// TODO: 实现算法
	return true
}
```

单元测试

```golang
func TestSensitiveWordFilterChain_Filter(t *testing.T) {
	chain := &SensitiveWordFilterChain{}
	chain.AddFilter(&AdSensitiveWordFilter{})
	assert.Equal(t, false, chain.Filter("test"))

	chain.AddFilter(&PoliticalWordFilter{})
	assert.Equal(t, true, chain.Filter("test"))
}
```




## 访问者模式 (Visitor)

- <font color='#808080'> 定义以及使用场景 </font>

由于没有函数重载，所以我们并不知道传递过来的对象是什么类型，这个时候只能采用类型断言的方式来对不同的类型做不同的操作，但是正式由于没有函数重载，所以其实完全可以不用访问者模式直接传入参数。

**使用场景**

访问者模式一共有五种角色：

(1) Vistor（抽象访问者）：为该对象结构中具体元素角色声明一个访问操作接口。
(2) ConcreteVisitor（具体访问者）：每个具体访问者都实现了Vistor中定义的操作。
(3) Element（抽象元素）：定义了一个accept操作，以Visitor作为参数。
(4) ConcreteElement（具体元素）：实现了Element中的accept()方法，调用Vistor的访问方法以便完成对一个元素的操作。


``` golang
package visitor

import (
	"fmt"
	"path"
)

// Visitor 访问者
type Visitor interface {
	Visit(IResourceFile) error
}

// IResourceFile IResourceFile
type IResourceFile interface {
	Accept(Visitor) error
}

// NewResourceFile NewResourceFile
func NewResourceFile(filepath string) (IResourceFile, error) {
	switch path.Ext(filepath) {
	case ".ppt":
		return &PPTFile{path: filepath}, nil
	case ".pdf":
		return &PdfFile{path: filepath}, nil
	default:
		return nil, fmt.Errorf("not found file type: %s", filepath)
	}
}

// PdfFile PdfFile
type PdfFile struct {
	path string
}

// Accept Accept
func (f *PdfFile) Accept(visitor Visitor) error {
	return visitor.Visit(f)
}

// PPTFile PPTFile
type PPTFile struct {
	path string
}

// Accept Accept
func (f *PPTFile) Accept(visitor Visitor) error {
	return visitor.Visit(f)
}

// Compressor 实现压缩功能
type Compressor struct{}

// Visit 实现访问者模式方法
// 我们可以发现由于没有函数重载，我们只能通过断言来根据不同的类型调用不同函数
// 但是我们即使不采用访问者模式，我们其实也是可以这么操作的
// 并且由于采用了类型断言，所以如果需要操作的对象比较多的话，这个函数其实也会膨胀的比较厉害
// 后续可以考虑按照命名约定使用 generate 自动生成代码
// 或者是使用反射简化
func (c *Compressor) Visit(r IResourceFile) error {
	switch f := r.(type) {
	case *PPTFile:
		return c.VisitPPTFile(f)
	case *PdfFile:
		return c.VisitPDFFile(f)
	default:
		return fmt.Errorf("not found resource typr: %#v", r)
	}
}

// VisitPPTFile VisitPPTFile
func (c *Compressor) VisitPPTFile(f *PPTFile) error {
	fmt.Println("this is ppt file")
	return nil
}

// VisitPDFFile VisitPDFFile
func (c *Compressor) VisitPDFFile(f *PdfFile) error {
	fmt.Println("this is pdf file")
	return nil
}
```

单元测试
``` golang
func TestCompressor_Visit(t *testing.T) {
	tests := []struct {
		name    string
		path    string
		wantErr string
	}{
		{
			name: "pdf",
			path: "./xx.pdf",
		},
		{
			name: "ppt",
			path: "./xx.ppt",
		},
		{
			name:    "404",
			path:    "./xx.xx",
			wantErr: "not found file type",
		},
	}

	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			f, err := NewResourceFile(tt.path)
			if tt.wantErr != "" {
				require.Error(t, err)
				require.Contains(t, err.Error(), tt.wantErr)
				return
			}

			require.NoError(t, err)
			compressor := &Compressor{}
			f.Accept(compressor)
		})
	}
}
```


## 迭代器模式(Iterator)

- <font color='#808080'> 定义以及使用场景 </font>

为了避免请求发送者与多个请求处理者耦合在一起，于是将所有请求的处理者通过前一对象记住其下一个对象的引用而连成一条链；当有请求发生时，可将请求沿着这条链传递，直到有对象处理它为止。


- **使用场景**

它在客户访问类与聚合类之间插入一个迭代器，这分离了聚合对象与其遍历行为，对客户也隐藏了其内部细节，且满足“单一职责原则”和“开闭原则”，如 Java 中的 Collection、List、Set、Map 等都包含了迭代器。


``` golang
// Iterator 迭代器接口
type Iterator interface {
	HasNext() bool
	Next()
	// 获取当前元素，由于 Go 1.15 中还没有泛型，所以我们直接返回 interface{}
	CurrentItem() interface{}
}

// ArrayInt 数组
type ArrayInt []int

// Iterator 返回迭代器
func (a ArrayInt) Iterator() Iterator {
	return &ArrayIntIterator{
		arrayInt: a,
		index:    0,
	}
}

// ArrayIntIterator 数组迭代
type ArrayIntIterator struct {
	arrayInt ArrayInt
	index    int
}

// HasNext 是否有下一个
func (iter *ArrayIntIterator) HasNext() bool {
	return iter.index < len(iter.arrayInt)-1
}

// Next 游标加一
func (iter *ArrayIntIterator) Next() {
	iter.index++
}

// CurrentItem 获取当前元素
func (iter *ArrayIntIterator) CurrentItem() interface{} {
	return iter.arrayInt[iter.index]
}
```


单元测试
``` golang
func TestArrayInt_Iterator(t *testing.T) {
	data := ArrayInt{1, 3, 5, 7, 8}
	iterator := data.Iterator()
	// i 用于测试
	i := 0
	for iterator.HasNext() {
		assert.Equal(t, data[i], iterator.CurrentItem())
		iterator.Next()
		i++
	}
}

```



# 其他

## 过滤器模式 (Pipe-filter)


- <font color='#808080'>定义以及使用场景</font>

对一数据需要经过顺序的多个过滤器函数处理。

- **使用场景**

1. 多个对象可以处理同一请求，其架构适用于 解析，过滤，处理，返回这样的架构，如数据分析。
2. 在请求处理者不明确的情况下向多个对象中的一个提交一个请求需要动态指定一组对象处理请求
<img style='height:320px' src='https://img1.kiosk007.top/static/images/design_pattern/pipe_filter.webp' />

- <font color='#200000'> 举个例子 </font>

下面的例子是一个将 字符串“1，2，3” 按逗号切分后，再字符转数字相加的过程。

``` bash
"1,2,3" --> [SplitFilter] --> ["1","2","3"] --> [ToIntFilter] --> [1,2,3] -> [SumFilter] --> 6
```

首先实现一个 filter 的接口。该接口定义了数据的来源接口，输出接口，该filter接口必须拥有的处理方法, 所有的过滤器必须参考这个接口实现。
``` go
// Request is the input of the filter
type Request interface{}

// Response is the output of the filter
type Response interface{}

// Filter interface is the definition of the data processing components
// Pipe-Filter structure
type Filter interface {
    Process(data Request) (Response, error)
}
```

实现SplitFilter，SplitFilter必须实现处理器Process。
``` go
var SplitFilterWrongFormatError = errors.New("input data should be string")

type SplitFilter struct {
    delimiter string
}

func NewSplitFilter(delimiter string) *SplitFilter {
    return &SplitFilter{delimiter}
}

func (sf *SplitFilter) Process(data Request) (Response, error) {
    str, ok := data.(string) //检查数据格式/类型，是否可以处理
    if !ok {
        return nil, SplitFilterWrongFormatError
    }
    parts := strings.Split(str, sf.delimiter)
    return parts, nil
}
```

实现ToIntFilter

``` go
var ToIntFilterWrongFormatError = errors.New("input data should be []string")

type ToIntFilter struct {
}

func NewToIntFilter() *ToIntFilter {
    return &ToIntFilter{}
}

func (tif *ToIntFilter) Process(data Request) (Response, error) {
    parts, ok := data.([]string)
    if !ok {
        return nil, ToIntFilterWrongFormatError
    }
    ret := []int{}
    for _, part := range parts {
        s, err := strconv.Atoi(part)
        if err != nil {
            return nil, err
        }
        ret = append(ret, s)
    }
    return ret, nil
}
```

实现 SumFilter

``` go
var SumFilterWrongFormatError = errors.New("input data should be []int")

type SumFilter struct {
}

func NewSumFilter() *SumFilter {
    return &SumFilter{}
}

func (sf *SumFilter) Process(data Request) (Response, error) {
    elems, ok := data.([]int)
    if !ok {
        return nil, SumFilterWrongFormatError
    }
    ret := 0
    for _, elem := range elems {
        ret += elem
    }
    return ret, nil
}
```

定义一个pipe-line， 目的是为了将所有的filter串起来。循环遍历整个filter并执行。

``` go
// NewStraightPipeline create a new StraightPipelineWithWallTime
func NewStraightPipeline(name string, filters ...Filter) *StraightPipeline {
    return &StraightPipeline{
        Name:    name,
        Filters: &filters,
    }
}

// StraightPipeline is composed of the filters, and the filters are piled as a straigt line.
type StraightPipeline struct {
    Name    string
    Filters *[]Filter
}

// Process is to process the coming data by the pipeline
func (f *StraightPipeline) Process(data Request) (Response, error) {
    var ret interface{}
    var err error
    for _, filter := range *f.Filters {
        ret, err = filter.Process(data)
        if err != nil {
            return ret, err
        }
        data = ret
    }
    return ret, err
}
```


执行

``` golang
func main() {
    spliter := pipe_filter.NewSplitFilter(",")
    converter := pipe_filter.NewToIntFilter()
    sum := pipe_filter.NewSumFilter()
    sp := pipe_filter.NewStraightPipeline("p1", spliter, converter, sum)
    ret, err := sp.Process("1,2,3")
    if err != nil {
        log.Fatal(err)
    }
    if ret != 6 {
        log.Fatalf("The expected is 6, but the actual is %d", ret)
    }
    fmt.Println(ret)
}

// 执行结果：
// 6

``` 





## 微内核模式 (Micro Kernel)

- <font color='#808080'> 定义以及使用场景 </font>

可以将 微核心架构理解成一个 核心要添加新功能就是加插件。其特点为 易扩展，错误隔离，保持架构的一致性。

**使用场景**

1. 如Nginx，启动前可以加载多个某块功能在应用上，不需要的时候可以剔除，但不影响整个应用的生命周期，适合一个应用的整体架构设计

- <font color='#200000'> 举个例子 </font>

**Agent**: agent 相当于一个注册中心，所有要Agent去做的事情都注册到Agent里面来，注册进Agent的操作叫做 Collector 。每个Collector有一个名字。用map存储。

``` go
type Agent struct {
    collectors map[string]Collector  //  注册进 Agent的collector
    evtBuf     chan Event            //  collector 回传给 Agent 的事件
    cancel     context.CancelFunc //  任务取消的方法
    ctx        context.Context    //  任务取消的上下文
    state      int                //  Agent 的运行状态
}
```

**Collector**: Collector 是一个执行器，是需要注册进上面的Agent的。每个Collector需要实现 Init，Start，Stop，Destroy 方法，到时候由 Agent 统一进行Init，Start等操作，这里在Init中提到了 EventReceiver，所有的Collector在初始化的时候传入一个事件接收源

``` go
type Collector interface {
    Init(evtReceiver EventReceiver) error    // Collector 将收集到的数据回传到 Agent （任何实现EventReceiver的对象）
    Start(agtCtx context.Context) error      // 启动所有的Collector（参数为agent中的取消上下文）
    Stop() error                              //   停止
    Destroy() error                           //   摧毁
}
```

**Event**: Agent 实现了 OnEvent 方法，所以Agent 可以作为上面Init 方法的参数，作为事件的接收者。


``` go
type Event struct {
    Source  string   // 事件源
    Content string   // 事件内容
}

type EventReceiver interface {
    OnEvent(evt Event)  // 实现OnEvent 既可以作为 EventReciver来接收事件 如下面的 Agent
}

func (agt *Agent) OnEvent(evt Event) {
    agt.evtBuf <- evt  // Agent 可以来接收事件
}
```

**开始起一个微内核**: 整个 微内核的架构就是这样了，刚才提到了，Agent会统一对注册进去的Collector进行初始化（Init），启动（Start），停止（Stop）的操作。 所以这里还差一个注册函数。

``` go
func (agt *Agent) RegisterCollector(name string, collector Collector) error {
    if agt.state != Waiting {
        return WrongStateError
    }
    agt.collectors[name] = collector   // agent map注册
    return collector.Init(agt)  // 注册完立即进行Init 操作。且事件接收者为Agent
}

```

**启动、停止、摧毁**
启动会将所有的Controller都拉起来，同理停止和摧毁。

``` go
func (agt *Agent) Start() error {
    if agt.state != Waiting {     // 状态不对，直接报错
        return WrongStateError
    }
    agt.state = Running    // 启动了，更改状态
    agt.ctx, agt.cancel = context.WithCancel(context.Background())  // 来一个取消的上下文和取消函数
    go agt.EventProcessGroutine()    // 收集事件 (具体业务了)
    return agt.startCollectors()    //  启动所有的Collector
}

// 模拟的收集具体的业务事件。这里的事件是由各个 collector 上报的
func (agt *Agent) EventProcessGroutine() {
    var evtSeg [10]Event
    for {
        for i := 0; i < 10; i++ {
            select {
            case evtSeg[i] = <-agt.evtBuf:   // 将 collector 收集的事件放到 evtBuf 中
            case <-agt.ctx.Done():           // 执行上下文完成，结束 
                return
            }
        }
        fmt.Println(evtSeg)   // 当 collector 收集的事件满 10 个，打印一次。
    }
}

// Agent 来拉起所有的 Collectors
func (agt *Agent) startCollectors() error {
    var err error
    var errs CollectorsError
    var mutex sync.Mutex
    for name, collector := range agt.collectors {
        go func(name string, collector Collector, ctx context.Context) {
            defer func() {
                mutex.Unlock()
            }()
            err = collector.Start(ctx)
            mutex.Lock()
            if err != nil {
                errs.CollectorErrors = append(errs.CollectorErrors,
                    errors.New(name+":"+err.Error()))
            }
        }(name, collector, agt.ctx)
    }
        if len(errs.CollectorErrors) == 0 {     // 这里需要判断有没有错误，确定没有错误，返回nil。否则其实返回的也不是nil
        return nil
    }
    return errs
}
```

**模拟**

``` go
type DemoCollector struct {      // 示例 Collector
    evtReceiver microkernel.EventReceiver   // 事件发给这里
    stopChan    chan struct{}    // 用来停止该Collector
    name        string    // Collector 名字
    content     string    // Collector 的要做的内容（假设，这个根据业务场景，都不一定是string）
}

func NewCollect(name string, content string) *DemoCollector {   // 创建一个 Collect
    return &DemoCollector{
        stopChan: make(chan struct{}),
        name:     name,
        content:  content,
    }
}

func (c *DemoCollector) Init(evtReceiver microkernel.EventReceiver) error {   // 初始化一个这个 Collect
    fmt.Println("initialize collector", c.name)
    c.evtReceiver = evtReceiver    // Agent 作为数据的上报源
    return nil
}

func (c *DemoCollector) Start(agtCtx context.Context) error {   // 拉起一个 Collect
    fmt.Println("start collector", c.name)

    for {    // 死循环
        select {
        case <-agtCtx.Done():      // 收到 Done 了
            c.stopChan <- struct{}{}  // 停掉该 Collect （Stop 方法那里会等 stopChan 这个信号）
            break
        default:
            time.Sleep(time.Millisecond * 50)
            c.evtReceiver.OnEvent(microkernel.Event{c.name, c.content}) // 向 Agent 上报事件
        }
    }

}

func (c *DemoCollector) Stop() error {
    fmt.Println("stop collector", c.name)
    select {
    case <-c.stopChan:   // 收到停止信号了，停掉
        return nil  // 一般停完再做点啥，在这做些善后吧
    case <-time.After(time.Second * 1):
        return errors.New("failed to stop for timeout")
    }
}

func (c *DemoCollector) Destroy() error {
    fmt.Println(c.name, "released resources.")
    return nil
}

func main() {
    var err error = nil
    agt := microkernel.NewAgent(100)
    c1 := NewCollect("c1", "1a")
    c2 := NewCollect("c2", "2b")
    if err = agt.RegisterCollector("c1", c1);err != nil {
        goto ERR
    }

    if err = agt.RegisterCollector("c2", c2);err != nil {
        goto ERR
    }

    if err = agt.Start();err != nil {
        goto ERR
    }

    time.Sleep(time.Second * 2)
    if err = agt.Stop();err != nil {
        goto ERR
    }

    return

ERR:
    fmt.Println("An Error Occur :",err)
}
```






