---
title: 实现一个Web服务-(一)
author: kiosk
draft: false
tags:
  - web
categories:
  - programming
date: 2022-04-09 10:15:00
---



熟悉 Golang 的话就会知道，用官方提供的 net/http 标准库搭建一个 Web Server，是一件非常简单的事。既然 Golang 利用其精简的语法实现了 Web Server，那么是有必要知其然和知其所以然。最近看到很多文章都介绍了如何实现一个 Web ，那么今天做一个总结。



<!--more-->

--------

- [实现一个Web服务-(二)]()



不同的框架设计和功能有很大的差别。任何一个爆款后端编程语言都会有一个属于她的Web实现，比如Python的 django、flask。同样的，Golang 的新框架也层出不穷，比如 Beego，Gin，Iris ，还有字节跳动出品的  hertz 。下面看看怎么实现一个 Web Server。

由于代码比较多，没有将每一步的操作都保存下来，不过也有很多出色的教程，比如 [Go语言动手写Web框架](https://geektutu.com/post/gee-day7.html)

最后的实现的git地址 :  https://github.com/kiosk404/goweb/



# 如何实现一个 Web Server 

Go 里可以使用 `net/http` 来创建一个 HTTP 服务快速实现一个简单的 Web Server。

``` go
// 创建一个bar路由和处理函数
http.HandleFunc("/bar", func(w http.ResponseWriter, r *http.Request) {
  fmt.Fprintf(w, "Hello, %q", html.EscapeString(r.URL.Path))
})

// 监听8080端口
log.Fatal(http.ListenAndServe(":8080", nil))
```

上面的例子中 `http.ListenAdnServe` 启动了一个 Web Server，`http.Handle`  创建了一个路由并注册了路由处理的函数，访问  http://127.0.0.1:8080/bar，可以得到 Hello, "/bar" 内容。



在 net/http 里，可以看到主要有以下这些结构。 

``` bash
go doc net/http | grep "^type"|grep struct
type Client struct{ ... }
type Cookie struct{ ... }
type ProtocolError struct{ ... }
type PushOptions struct{ ... }
type Request struct{ ... }
type Response struct{ ... }
type ServeMux struct{ ... }
type Server struct{ ... }
type Transport struct{ ... }
```



- Client 负责构建 HTTP 客户端；
- Server 负责构建 HTTP 服务端；
- ServerMux 负责 HTTP 服务端路由；
- Transport、Request、Response、Cookie 负责客户端和服务端传输对应的不同模块。



## HTTP 基础

在 `net/http.Server` 这个结构体中最重要的是 Handler，当 `ListenAndServe(addr string, handler Handler)` 中的Handler 这个字段为空的时候，他默认会用 `DefaultServerMux` 这个路由器来填充。通过查看源码可以发现 Handler 是一个接口，需要实现方法 **`ServeHTTP`**, 也就是传入了任何实现了 ServerHTTP 接口的实例，就能处理一个传入的HTTP请求。



创建一个 framework 文件夹，新建 `core.go`, 在里面写入如下内容。

```go
// 框架核心结构
type Core struct {
}

// 初始化框架核心结构
func NewCore() *Core {
	return &Core{}
}

// 框架核心结构实现Handler接口
func (c *Core) ServeHTTP(response http.ResponseWriter, request *http.Request) {
	switch request.URL.Path {
	case "/":
		fmt.Fprintf(response, "URL.Path = %q\n", request.URL.Path)
	case "/hello":
		for k, v := range request.Header {
			fmt.Fprintf(response, "Header[%q] = %q\n", k, v)
		}
	default:
		fmt.Fprintf(response, "404 NOT FOUND: %s\n", request.URL)
	}
}

```



这下我们在 `main.go` 的里面调用做如下替换，即完成了一个最基础的 Web Server。但是这里的路由使用一个 switch 语句实现，还比较原始，接下来需要再对 **路由** 进行封装。

```go
server := http.Server{
		Addr:              ":8080",
		Handler:           framework.NewCore(),
	}
// 监听8080端口
server.ListenAndServe()
```



不管怎么说，肯定是要对 `Core` 结构体进行改造了，我们将路由规则由 switch 匹配的模式改为 map 存储的格式，先实现一个**静态路由匹配**。实现最基本的功能。



```go
// HandlerFunc defines the request handler used by gee
type HandlerFunc func(http.ResponseWriter, *http.Request)

// 框架核心结构
type Core struct {
	router map[string]HandlerFunc
}

func (c *Core) registerRouter(method, pattern string, handlerFunc HandlerFunc) {
	key := method + "-" + pattern
	c.router[key] = handlerFunc
}

func (c *Core) Get(url string, handler HandlerFunc) {
	c.registerRouter("GET", url, handler)
}

func (c *Core) Post(url string, handler HandlerFunc) {
	c.registerRouter("POST", url, handler)
}

// 初始化框架核心结构
func NewCore() *Core {
	return &Core{
		router: make(map[string]HandlerFunc),
	}
}

// 框架核心结构实现Handler接口
func (c *Core) ServeHTTP(response http.ResponseWriter, request *http.Request) {
	key := request.Method + "-" + request.URL.Path
	if handler, ok := c.router[key]; ok {
		handler(response, request)
	} else {
		fmt.Fprintf(response, "404 NOT FOUND: %s\n", request.URL)
	}
}

```



这下和就和标准的 HTTP Web框架比较像了，在 main 函数里进行路由注册。

``` go
func main() {
    core := framework.NewCore()
	core.Get("/foo", func(w http.ResponseWriter, req *http.Request) {
		fmt.Fprintf(w, "URL.Path = %q\n", req.URL.Path)
	})

	server := http.Server{
		Addr:              ":8080",
		Handler:           core,
	}
	// 监听8080端口
	server.ListenAndServe()
}
```



## 上下文

http 里的上下文不仅是为了封装 Request 和 Response，向外提供 `JSON` 或者 `HTML` 的返回，另外还肩负者请求的超时处理等工作。我们来分析一下 上下文 设计的必要性。

1. 在 **[beego](https://beego.vip/docs/mvc/controller/controller.md)** 中的效果，我们还需要对一些模板、常见返回格式封装，比如我们在返回 json 时，需要对 Response Header 中的 Content-Type 字段进行设定，而一个优秀的 Web Context 就需要对其进行封装，避免大量重复、复杂的代码。比如一个返回码为 <font style="color:green">200</font> 的JSON返回，经过封装如下即可。

``` go
c.JSON(http.StatusOK, gee.H{
    "username": c.PostForm("username"),
    "password": c.PostForm("password"),
})
```



2. 官方库里可以看到其主要实现了 取消、超时、截止时间的Context。

``` go
go doc context | grep "^func" 
func WithCancel(parent Context) (ctx Context, cancel CancelFunc)
func WithDeadline(parent Context, d time.Time) (Context, CancelFunc)
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc)
```

熟悉 `beego` 的同学可能都经常用创建 Context 的方式封装一个 Controller 。**每个连接的 Context 都基于 baseContext 复制而来。** 对应到代码中，就是在某个连接开启 Goroutine 的时候，为当前的连接创建了一个 connContext，这个 connContext 是基于 server 中的 Context 而来，而 server 中的 Context 的基础就是 baseContext。

```go

type Server struct {
  ...

    // BaseContext 用来为整个链条创建初始化 Context
    // 如果没有设置的话，默认使用 context.Background()
  BaseContext func(net.Listener) context.Context{}

    // ConnContext 用来为每个连接封装 Context
    // 参数中的 context.Context 是从 BaseContext 继承来的
  ConnContext func(ctx context.Context, c net.Conn) context.Context{}
    ...
}
```



3.  除此之外，一个框架肯定是需要支持中间件的，中间件产生的信息需要交给上下文去处理，Context 随着一个请求的出现而产生，请求的结束而销毁。



为了实现 Context 对现有 Core 的替换，我们新创建2个文件`context.go` 和 `controller.go` 。其中 controller.go 用来将我们的 handler 封装过度到 context。

``` go
// controller.go

type ControllerHandler func(c *Context) error
```



context 里的内容比较多，主要做的事情就是将 `ctx.request.Context()` 暴露出来和对返回内容的封装

```go
// context.go

type Context struct {
	request        *http.Request
	responseWriter http.ResponseWriter
	ctx            context.Context
	handler        ControllerHandler

	// 是否超时标记位
	hasTimeout bool
	// 写保护机制
	writerMux *sync.Mutex
}

func NewContext(r *http.Request, w http.ResponseWriter) *Context {
	return &Context{
		request:        r,
		responseWriter: w,
		ctx:            r.Context(),
		writerMux:      &sync.Mutex{},
	}
}

// #region base function

func (ctx *Context) WriterMux() *sync.Mutex {
	return ctx.writerMux
}

func (ctx *Context) GetRequest() *http.Request {
	return ctx.request
}

func (ctx *Context) GetResponse() http.ResponseWriter {
	return ctx.responseWriter
}

func (ctx *Context) SetHasTimeout() {
	ctx.hasTimeout = true
}

func (ctx *Context) HasTimeout() bool {
	return ctx.hasTimeout
}

// #endregion

func (ctx *Context) BaseContext() context.Context {
	return ctx.request.Context()
}

// #region implement context.Context
func (ctx *Context) Deadline() (deadline time.Time, ok bool) {
	return ctx.BaseContext().Deadline()
}

func (ctx *Context) Done() <-chan struct{} {
	return ctx.BaseContext().Done()
}

func (ctx *Context) Err() error {
	return ctx.BaseContext().Err()
}

func (ctx *Context) Value(key interface{}) interface{} {
	return ctx.BaseContext().Value(key)
}

// #endregion

// #region response

func (ctx *Context) Json(status int, obj interface{}) error {
	if ctx.HasTimeout() {
		return nil
	}
	ctx.responseWriter.Header().Set("Content-Type", "application/json")
	ctx.responseWriter.WriteHeader(status)
	byt, err := json.Marshal(obj)
	if err != nil {
		ctx.responseWriter.WriteHeader(500)
		return err
	}
	ctx.responseWriter.Write(byt)
	return nil
}

func (ctx *Context) HTML(status int, obj interface{}, template string) error {
	return nil
}

func (ctx *Context) Text(status int, obj string) error {
	ctx.responseWriter.Header().Set("Content-Type", "application/text")
	ctx.responseWriter.WriteHeader(status)
	_, err := ctx.responseWriter.Write([]byte(obj))
	return err
}

// #endregion
```



如此以来，将 core.go 的封装也改动一下 , 我们 Context 的改造也完毕了。



``` go
// 框架核心结构
type Core struct {
	router map[string]ControllerHandler
}

func (c *Core) Get(url string, handler ControllerHandler) {
	c.registerRouter("GET", url, handler)
}

func (c *Core) Post(url string, handler ControllerHandler) {
	c.registerRouter("POST", url, handler)
}

// 初始化框架核心结构
func NewCore() *Core {
	return &Core{
		router: make(map[string]ControllerHandler),
	}
}

// 框架核心结构实现Handler接口
func (c *Core) ServeHTTP(response http.ResponseWriter, request *http.Request) {
	ctx := NewContext(request, response)
	key := request.Method + "-" + request.URL.Path

	if handler, ok := c.router[key]; ok {
		handler(ctx)
	} else {
		ctx.Text(http.StatusNotFound, fmt.Sprintf("404 NOT FOUND: %s\n", request.URL.Path))
	}
}
```



## 路由

一个Web服务器框架的路由主要需要考虑 **HTTP 方法匹配**、**静态路由匹配**、**批量通用前缀**、**动态路由匹配** 。

这里的路由就应该考虑一下复杂的逻辑了，比如动态匹配（Dynamic Route） 和 分组控制（Group Control）。

### **动态匹配 - （前缀树 Trie）**

之前我们是以一个简单的 Map 结构存储了路由表，但是这就导致我们的路由只能是静态的。比如我们如何支持类似于 "/v1/hello/:name" 这样的路由呢？这种路由下 "/v1/hello/kiosk" 或者 "、"/v1/hello/bob" 都是可以的。



动态路由有很多实现方式，支持的规则、性能等有很大的差异，开源的实现有 **"[gorouter](https://github.com/cloudfoundry/gorouter)"** 。支持包括正则表达式的匹配，另一个开源版本 **[httprouter](https://github.com/julienschmidt/httprouter)** 则不支持正则。



Trie 树具备参数匹配：例如 "/p/:lang/doc", 可以匹配 "/p/c/doc" 和 "/p/go/doc"



**Trie 实现**

创建文件 trie.go , 实现树节点存储的信息

``` go
// 代表树结构
type Tree struct {
	root *node // 根节点
}

// 代表节点
type node struct {
	isLast  bool              // 该节点是否能成为一个独立的uri, 是否自身就是一个终极节点
	segment string            // uri中的字符串
	handler ControllerHandler // 控制器
	childs  []*node           // 子节点
}

```

与普通的树不同，为了实现动态路由，我们需要每一层都匹配，`isWildSegment` 判断是否范匹配到 ":xxx"。每次将匹配剩下的字符串拆成两份，如果拆成2份说明，没匹配完，拆成一份则表示匹配完了，现在这个 segment 是最后一个。



``` go
// 判断路由是否已经在节点的所有子节点树中存在了
func (n *node) matchNode(uri string) *node {
    // 使用分隔符将uri切割为两个部分, 如 "/api/v1" 会被拆成 ""(/前面是空) 和 "api/v1"
	segments := strings.SplitN(uri, "/", 2)
	// 第一个部分用于匹配下一层子节点
	segment := segments[0] 
	segment = strings.ToUpper(segment)

	// 匹配符合的下一层子节点
	cnodes := n.filterChildNodes(segment)
	// 如果当前子节点没有一个符合，那么说明这个uri一定是之前不存在, 直接返回nil
	if cnodes == nil || len(cnodes) == 0 {
		return nil
	}

	// 如果只有一个segment，则是最后一个标记
	if len(segments) == 1 {
		// 如果segment已经是最后一个节点，判断这些cnode是否有isLast标志
		for _, tn := range cnodes {
			if tn.isLast {
				return tn
			}
		}

		// 都不是最后一个节点
		return nil
	}

	// 如果有2个segment, 递归每个子节点继续进行查找
	for _, tn := range cnodes {
		tnMatch := tn.matchNode(segments[1])
		if tnMatch != nil {
			return tnMatch
		}
	}
	return nil
}



// 过滤下一层满足segment规则的子节点
func (n *node) filterChildNodes(segment string) []*node {
	if len(n.childs) == 0 {
		return nil
	}

	// 如果segment是通配符，则所有下一层子节点都满足需求
	if isWildSegment(segment) {
		return n.childs
	}

	nodes := make([]*node, 0, len(n.childs))
	// 过滤所有的下一层子节点
	for _, cnode := range n.childs {
		if isWildSegment(cnode.segment) {
			// 如果下一层子节点有通配符，则满足需求
			nodes = append(nodes, cnode)
		} else if cnode.segment == segment {
			// 如果下一层子节点没有通配符，但是文本完全匹配，则满足需求
			nodes = append(nodes, cnode)
		}
	}

	return nodes
}
```



第10行在当前Node 节点遍历出所有符合的 node ，再到第30行递归接着遍历。直到遍历到最后segment split 结果为 1，返回最终的那个node。



除了查找之外，前缀树另外一个比较重要的部分是插入一个路由。

``` go
func (tree *Tree) AddRouter(uri string, handler ControllerHandler) error {
	n := tree.root
	if n.matchNode(uri) != nil {
		return errors.New("route exist: " + uri)
	}

	segments := strings.Split(uri, "/")
	// 对每个segment
	for index, segment := range segments {

		// 最终进入Node segment的字段
		segment = strings.ToUpper(segment)
		isLast := index == len(segments)-1

		var objNode *node // 标记是否有合适的子节点

		childNodes := n.filterChildNodes(segment)
		// 如果有匹配的子节点
		if len(childNodes) > 0 {
			// 如果有segment相同的子节点，则选择这个子节点
			for _, cnode := range childNodes {
				if cnode.segment == segment {
					objNode = cnode
					break
				}
			}
		}

		if objNode == nil {
			// 创建一个当前node的节点
			cnode := newNode()
			cnode.segment = segment
			if isLast {
				cnode.isLast = true
				cnode.handler = handler
			}
			n.childs = append(n.childs, cnode)
			objNode = cnode
		}

		n = objNode
	}

	return nil
}

```



如此我们即实现了一个前缀树匹配路由。

### **分组控制**

分组控制(Group Control)是 Web 框架应提供的基础功能之一。所谓分组，是指路由的分组。如果没有路由分组，我们需要针对每一个路由进行控制。但是真实的业务场景中，往往某一组路由需要相似的处理。例如：某一批的API均已 "/api/v1" 开头。



``` go
// IGroup 代表前缀分组
type IGroup interface {
	// 实现HttpMethod方法
	Get(string, ControllerHandler)
	Post(string, ControllerHandler)

	// 实现嵌套group
	Group(string) IGroup
}

// Group struct 实现了IGroup
type Group struct {
	core   *Core  // 指向core结构
	parent *Group //指向上一个Group，如果有的话
	prefix string // 这个group的通用前缀
}

// 初始化Group
func NewGroup(core *Core, prefix string) *Group {
	return &Group{
		core:   core,
		parent: nil,
		prefix: prefix,
	}
}
```



一个 Group 需要实现首先指向 core 的指针。另外还需要 父节点的分组（如果有的话）, 另外就是最基础的当前这个通用前缀的名字了，另外他需要实现 Core 的 Get、Post 等方法。



生成一个 Group 的方法实现即

``` go
// 实现 Group 方法
func (g *Group) Group(uri string) IGroup {
	cgroup := NewGroup(g.core, uri)
	cgroup.parent = g
	return cgroup
}
```



如果想要在这个Group上加方法的话，则可以

``` go
// 实现Get方法
func (g *Group) Get(uri string, handler ControllerHandler) {
	uri = g.getAbsolutePrefix() + uri
	g.core.Get(uri, handler)
}

// 实现Post方法
func (g *Group) Post(uri string, handler ControllerHandler) {
	uri = g.getAbsolutePrefix() + uri
	g.core.Post(uri, handler)
}

// 获取当前group的绝对路径
func (g *Group) getAbsolutePrefix() string {
	if g.parent == nil {
		return g.prefix
	}
	return g.parent.getAbsolutePrefix() + g.prefix
}
```

如此一来，我们可以对 `framework.Core` 进行改造了。之前的静态 map 路由的方式可以改掉了，也不需要 key 由 “Method” 和 “URL”  拼接生成了，而是可以用 “Method” 当key，所有和路由相关的内容全部放入到 **Trie Tree** 里。core.go 改造如下：

```go
// 框架核心结构
type Core struct {
	router map[string]*Tree
}

func (c *Core) Get(url string, handler ControllerHandler) {
	if err := c.router["GET"].AddRouter(url, handler); err != nil {
		log.Fatal("add router error: ", err)
	}
}

func (c *Core) Post(url string, handler ControllerHandler) {
	if err := c.router["POST"].AddRouter(url, handler); err != nil {
		log.Fatal("add router error: ", err)
	}
}

// 初始化框架核心结构
func NewCore() *Core {
	router := map[string]*Tree{}
	router["GET"] = NewTree()
	router["POST"] = NewTree()
	return &Core{router: router}
}

// ==== http method wrap end

func (c *Core) Group(prefix string) IGroup {
	return NewGroup(c, prefix)
}

// 匹配路由，如果没有匹配到，返回nil
func (c *Core) FindRouteByRequest(request *http.Request) ControllerHandler {
	// uri 和 method 全部转换为大写，保证大小写不敏感
	uri := request.URL.Path
	method := request.Method
	upperMethod := strings.ToUpper(method)

	// 查找第一层map
	if methodHandlers, ok := c.router[upperMethod]; ok {
		return methodHandlers.FindHandler(uri)
	}
	return nil
}

// 框架核心结构实现Handler接口
func (c *Core) ServeHTTP(response http.ResponseWriter, request *http.Request) {
	ctx := NewContext(request, response)
	// 寻找路由
	router := c.FindRouteByRequest(request)
	if router == nil {
		// 如果没有找到，这里打印日志
		_ = ctx.Json(404, "not found")
		return
	}

	// 调用路由函数，如果返回err 代表存在内部错误，返回500状态码
	if err := router(ctx); err != nil {
		_ = ctx.Json(500, "inner error")
		return
	}
}
```



至此完整的代码可以参考： https://github.com/kiosk404/goweb/tree/web_router



## 中间件

中间件(middlewares)，简单说，就是非业务的技术类组件。Web 框架本身不可能去理解所有的业务，因而不可能实现所有的功能。因此，框架需要有一个插口，允许用户自己定义功能，嵌入到框架中，仿佛这个功能是框架原生支持的一样。



之前其实就提到过了，中间件的实现离不开上面已经实现了的 `Context` 。我们处理的思路就是 **PipeLine 改造中间件**，使用一个数组链接起来，形成一个流水线，就完美的解决了这个问题。

请求流如下：

![goweb_middleware](https://img1.kiosk007.top/static/images/blog/goweb_middleware.png)



中间件的插入点是框架接收到请求初始化 Context 对象后，允许用户做一写自己的事情，如记录日志等，完成后，调用 `(*Context).Next()` 函数。

举个例子：

``` go
func Logger() ControllerHandler {
    return func(c *Context) {
		// Start timer
		t := time.Now()
		// Process request
		c.Next()
		// Calculate resolution time
		log.Printf("[%d] %s in %v", c.StatusCode, c.Req.RequestURI, time.Since(t))
	}
}
```



好了，开始对 Context 进行相关的改造：

``` go
// Context代表当前请求上下文
type Context struct {
	request        *http.Request
	responseWriter http.ResponseWriter
	ctx            context.Context
	writerMux  *sync.Mutex

	// 当前请求的handler链条
	handlers []ControllerHandler
	index    int // 当前请求调用到调用链的哪个节点
}

// NewContext 初始化一个Context
func NewContext(r *http.Request, w http.ResponseWriter) *Context {
	return &Context{
		request:        r,
		responseWriter: w,
		ctx:            r.Context(),
		writerMux:      &sync.Mutex{},
		index:          -1,
	}
}
```



接下来就是最核心的函数 `Next()`  ，在中间件中调用 Next 方法来执行下一个 Handler 内容。

``` go
// 核心函数，调用context的下一个函数
func (ctx *Context) Next() error {
	ctx.index++
	if ctx.index <= len(ctx.handlers) {
		if err := ctx.handlers[ctx.index-1](ctx); err != nil {
			return err
		}
	}
	return nil
}

```



当然 core 也就需要相应的改造了，首先是前缀树路由，路由不再是一个 Handler ，而是一个 Handler 列表。另外我们还需要保存所有已经注册的中间件。



``` go
// 框架核心结构
type Core struct {
	router map[string]*Tree
	middlewares []ControllerHandler // 从core这边设置的中间件
}

// 注册中间件
func (c *Core) Use(middlewares ...ControllerHandler) {
	c.middlewares = append(c.middlewares, middlewares...)
}
```



相应的，如果添加 Method 方法，我们需要先将中间件加进去。

``` go
func (c *Core) Get(url string, handlers ...ControllerHandler) {
	// 将core的middleware 和 handlers结合起来
	allHandlers := append(c.middlewares, handlers...)
	if err := c.router["GET"].AddRouter(url, allHandlers); err != nil {
		log.Fatal("add router error: ", err)
	}
}
```



如此以来，`ServeHTTP()` 方法就需要小小的改造，封装好一个 Context ，从路由表里找到 handlers 列表，将 handlers 加入到 ctx 中依次遍历执行。



``` go
// 框架核心结构实现Handler接口
func (c *Core) ServeHTTP(response http.ResponseWriter, request *http.Request) {
	ctx := NewContext(request, response)
	// 寻找路由
	handlers := c.FindRouteByRequest(request)
	if handlers == nil {
		// 如果没有找到，这里打印日志
		_ = ctx.Json(404, "not found")
		return
	}

	ctx.SetHandlers(handlers)

	// 调用路由函数，如果返回err 代表存在内部错误，返回500状态码
	if err := ctx.Next(); err != nil {
		ctx.Json(500, "inner error")
		return
	}
}
```



下面写几个中间件，一个是计算一个请求的耗时，你个是设置请求的超时时间。

```go
// 超时的中间件
func Timeout(d time.Duration) framework.ControllerHandler {
	// 使用函数回调
	return func(c *framework.Context) error {
		finish := make(chan struct{}, 1)
		panicChan := make(chan interface{}, 1)
		// 执行业务逻辑前预操作：初始化超时context
		durationCtx, cancel := context.WithTimeout(c.BaseContext(), d)
		defer cancel()

		go func() {
			defer func() {
				if p := recover(); p != nil {
					panicChan <- p
				}
			}()
			// 使用next执行具体的业务逻辑
			c.Next()

			finish <- struct{}{}
		}()
		// 执行业务逻辑后操作
		select {
		case p := <-panicChan:
			c.Json(500, "server internal error")
			log.Println(p)
		case <-finish:
			fmt.Println("finish")
		case <-durationCtx.Done():
			fmt.Println("超时了")
			c.SetHasTimeout()
			c.Json(http.StatusBadGateway, "time out")
		}
		return nil
	}
}

```



统计时间的中间件

``` go
// 统计运行时间的中间件
func Cost() framework.ControllerHandler {
	// 使用函数回调
	return func(c *framework.Context) error {
		// 记录开始时间
		start := time.Now()

		// 使用next执行具体的业务逻辑
		c.Next()

		// 记录结束时间
		end := time.Now()
		cost := end.Sub(start)
		fmt.Printf("api uri: %v, cost: %v \n", c.GetRequest().RequestURI, cost.Seconds())

		return nil
	}
}
```



相关的代码：https://github.com/kiosk404/goweb/tree/middleware



# 总结

至此，一个 WebServer 的 MVP版本就诞生了，其拥有 前缀树路由匹配、分组功能、加载中间件。其核心的功能都已经完成，其余的功能最多是丰富内容锦上添花。后面会介绍一个 WebServer 的更高阶内容。