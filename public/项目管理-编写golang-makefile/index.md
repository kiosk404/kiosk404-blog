# 项目管理--编写Golang MakeFile

Makefile 可以简单的认为是一个工程文件的编译规则，描述了整个工程的编译和链接等规则。其中包含了那些文件需要编译，那些文件不需要编译，那些文件需要先编译，那些文件需要后编译，那些文件需要重建等等。编译整个工程需要涉及到的，在 Makefile 中都可以进行描述。换句话说，Makefile 可以使得我们的项目工程的编译变得自动化，不需要每次都手动输入一堆源文件和参数。


<!--more-->


以 Linux 下的C语言开发为例来具体说明一下，多文件编译生成一个文件，编译的命令如下所示：

``` bash
$ gcc -o outfile name1.c name2.c ...

```
outfile 要生成的可执行程序的名字，nameN.c 是源文件的名字。

下面列举了一些需要我们手动链接的标准库：
- name1.c 用到了数学计算库 math 中的函数，我们得手动添加参数 -Im；
- name4.c 用到了小型数据库 SQLite 中的函数，我们得手动添加参数 -lsqlite3；
- name5.c 使用到了线程，我们需要去手动添加参数 -lpthread。

# Make 语法

编写高质量 Makefile 的第一步，便是熟练掌握 Makefile 的核心语法。
这里可以参考 [《跟我一起写Makefile (PDF重制版)》](https://github.com/seisman/how-to-write-makefile)、[Makefile 教程](http://c.biancheng.net/view/7097.html)

makefile 完整案例[IAM Makefile](https://github.com/marmotedu/iam/blob/master/Makefile)


makefile 的基本写法如下图，由 `target`,`dependencies`,`command`（command前必须是 tab 键） 组成
<img src="https://img1.kiosk007.top/static/images/makefile/grammar1.png">


- `target`：规则的目标，可以是 Object File（一般称它为中间文件），也可以是可执行文件，还可以是一个标签；
- `dependencies`：是我们的依赖文件，要生成 targets 需要的文件或者是目标。可以是多个，也可以是没有；
- `command`：make 需要执行的命令（任意的 shell 命令）。可以有多条命令，每一条命令占一行。

一个最基本的 golang Makefile 文件

``` makefile
build: clean vet
  @mkdir -p ./Role
  @export GOOS=linux && go build -v .

vet:
  go vet ./...

fmt:
  go fmt ./...

clean:
  rm -rf dashboard
```
> shell 命令行前的 @ 是为了防止回显，否则会将 `mkdir -p ./Role`输出出来再执行


``` makefile
# ==============================================================================
# Usage

define USAGE_OPTIONS

Options:
  DEBUG        Whether to generate debug symbols. Default is 0.
  BINS         The binaries to build. Default is all of cmd.
               This option is available when using: make 
endef
export USAGE_OPTIONS

## build: Build source code for host platform.
.PHONY: build
build:
	@$(MAKE) go.build


## help: Show this help info.
.PHONY: help
help: Makefile
	@echo -e "\nUsage: make <TARGETS> <OPTIONS> ...\n\nTargets:"
	@sed -n 's/^##//p' $< | column -t -s ':' | sed -e 's/^/ /'
	@echo "$$USAGE_OPTIONS"
```
这个进阶的Makefile有以下知识点

## 伪目标

所谓的伪目标可以这样来理解，它并不会创建目标文件，只是想去执行这个目标下面的命令。伪目标的存在可以帮助我们找到命令并执行。

特殊的标记 “.PHONY” 来显式地指明一个目标是“伪目标”，“.PHONY” 标记的target可以理解成一个无条件执行的动作。
这样就可以执行 `make help` 命令了。

使用`伪目标`主要有2个原因：
1. 避免我们的 Makefile 中定义的只执行的命令的目标和工作目录下的实际文件出现名字冲突。
2. 提高执行 make 时的效率，特别是对于一个大型的工程来说，提高编译的效率也是我们所必需的。



如下命令：

``` makefile
clean:
    rm -rf *.o test
```

规则中 rm 命令不是创建文件 clean 的命令，而是执行删除任务，删除当前目录下的所有的 .o 结尾和文件名为 test 的文件。当工作目录下不存在以 clean 命令的文件时，在 shell 中输入 make clean 命令，命令 rm -rf *.o test 总会被执行 ，这也是我们期望的结果。

如果当前目录下存在文件名为  clean 的文件时情况就会不一样了，当我们在 shell 中执行命令 make clean，由于这个规则没有依赖文件，所以目标被认为是最新的而不去执行规则所定义的命令。因此命令 rm 将不会被执行。为了解决这个问题，删除 clean 文件或者是在 Makefile 中将目标 clean 声明为伪目标。将一个目标声明称伪目标的方法是将它作为特殊的目标.PHONY的依赖，比如 `.PHONY:clean`


## 自动化变量 (Automatic Variables）
help 后面跟了一个文件 `Makefile`, 没错！就是help会将`Makefile` 当成参数。help 后面里有几个 `$$`、`$<` 变量

`$<` 可以引用 Makefile 当成参数。没错，上述的命令展开就是 `sed -n 's/^##//p' Makefile | column -t -s ':' | sed -e 's/^/ /'`。
其实是将整个 `Makefile` 中的双井号解析成帮助命令。


| 自动化变量       | 说明   |
| :--------   |:----- |
|\$@|表示规则的目标文件名。如果目标是一个文档文件（Linux 中，一般成.a文件为文档文件，也成为静态的库文件），那么它代表这个文档的文件名。在多目标模式规则中，它代表的是触发规则被执行的文件名。 |
|\$%|当目标文件是一个静态库文件时，代表静态库的一个成员名。|
|\$<|规则的第一个依赖的文件名。如果是一个目标文件使用隐含的规则来重建，则它代表由隐含规则加入的第一个依赖文件。 |
|\$?|所有比目标文件更新的依赖文件列表，空格分隔。如果目标文件时静态库文件，代表的是库文件（.o 文件）。|
|\$^| 代表的是所有依赖文件列表，使用空格分隔。如果目标是静态库文件，它所代表的只能是所有的库成员（.o 文件）名。一个文件可重复的出现在目标的依赖中，变量“$^”只记录它的第一次引用的情况。就是说变量“$^”会去掉重复的依赖文件|
|\$+|类似“$^”，但是它保留了依赖文件中重复出现的文件。主要用在程序链接时库的交叉引用场合。|

举个例子：

```makefile
test:test.o test1.o test2.o
         gcc -o $@ $^
test.o:test.c test.h
         gcc -o $@ $<
test1.o:test1.c test1.h
         gcc -o $@ $<
test2.o:test2.c test2.h
         gcc -o $@ $<
```

这个规则模式中用到了 "$@" 、"$<" 和 "$^" 这三个自动化变量，对比之前写的 Makefile 中的命令，我们可以发现
- "$@" 代表的是目标文件test
- “$^”代表的是依赖的文件
- “$<”代表的是依赖文件中的第一个。

我们在执行 make 的时候，make 会自动识别命令中的自动化变量，并自动实现自动化变量中的值的替换，这个类似于编译C语言文件的时候的预处理的作用。



## 内置变量（Implicit Variables）
上面的实例中有这样一个表达式 

```makefile
## build: Build source code for host platform.
.PHONY: build
build:
	@$(MAKE) go.build
```

Make命令提供一系列内置变量，（感觉上和gcc/g++的预定义宏差不多）比如，`$(CC)` 指向当前使用的编译器，`$(MAKE)` 指向当前使用的Make工具。这主要是为了跨平台的兼容性，详细的内置变量清单见[手册](https://www.gnu.org/software/make/manual/html_node/Implicit-Variables.html)。

-----

看下面的例子，其是一个 `common.mk` 提供一些 Makefile 文件调用时的基础变量环境。


```makefile

SHELL := /bin/bash

# include the common make file
COMMON_SELF_DIR := $(dir $(lastword $(MAKEFILE_LIST)))

ifeq ($(origin ROOT_DIR),undefined)
ROOT_DIR := $(abspath $(shell cd $(COMMON_SELF_DIR)/../.. && pwd -P))
endif
ifeq ($(origin OUTPUT_DIR),undefined)
OUTPUT_DIR := $(ROOT_DIR)/_output
$(shell mkdir -p $(OUTPUT_DIR))
endif
ifeq ($(origin TOOLS_DIR),undefined)
TOOLS_DIR := $(OUTPUT_DIR)/tools
$(shell mkdir -p $(TOOLS_DIR))
endif
ifeq ($(origin TMP_DIR),undefined)
TMP_DIR := $(OUTPUT_DIR)/tmp
$(shell mkdir -p $(TMP_DIR))
endif

```

上面的例子有以下几个知识点，`:=`、`ifeq`、`$()` 等。

## 变量
先说变量，变量对于我们来说是不陌生的，在学习各种编程语言时会经常用到。就拿C语言来说，变量的使用是十分常见的，变量可以用来保存一个值或者是使用变量进行运算操作。

调用变量的时候可以用 "$(VALUE_LIST)" 或者是 "${VALUE_LIST}" 来替换，这就是变量的引用。实例：
```makefile
OBJ=main.o test.o test1.o test2.o
test:$(OBJ)
      gcc -o test $(OBJ)
```

**变量的基本赋值**
知道了如何定义，下面我们来说一下 Makefile 的变量的四种基本赋值方式：
- 简单赋值 ( := ) 编程语言中常规理解的赋值方式，只对当前语句的变量有效。
- 递归赋值 ( = ) 赋值语句可能影响多个变量，所有目标变量相关的其他变量都受影响。
- 条件赋值 ( ?= ) 如果变量未定义，则使用符号中的值定义变量。如果该变量已经赋值，则该赋值语句无效。
- 追加赋值 ( += ) 原变量用空格隔开的方式追加一个新值。

其中的简单赋值和条件赋值比较简单字面意思就可以理解，下面介绍一下另外两种。
**递归赋值**

``` makefile
x=foo
y=$(x)b
x=new
test：
      @echo "y=>$(y)"
      @echo "x=>$(x)"
## 执行结果
## y=>newb
## x=>new
```

**追加赋值**

``` makefile
x:=foo
y:=$(x)b
x+=$(y)
test：
      @echo "y=>$(y)"
      @echo "x=>$(x)"
## 执行结果
## y=>foob
## x=>foo foob
```


## 条件语句
条件语句只能用于控制 make 实际执行的 Makefile 文件部分，不能控制规则的 shell 命令执行的过程。

|关键字|说明|
|:----|:----|
|ifeq|判断参数是否不相等，相等为 true，不相等为 false|
|ifneq|判断参数是否不相等，不相等为 true，相等为 false|
|ifeq|判断是否有值，有值为 true，没有值为 false|
|ifneq|判断是否有值，没有值为 true，有值为 false|





# Makefile 要实现的功能

对于 Go 项目来说，虽然不同项目集成的功能不一样，但绝大部分项目都需要实现一些通用的功能。接下来，我们就来看看，在一个大型 Go 项目中 Makefile 通常可以实现的功能。

``` bash

$ make help

Usage: make <TARGETS> <OPTIONS> ...

Targets:
  # 代码生成类命令
  gen                Generate all necessary files, such as error code files.

  # 格式化类命令
  format             Gofmt (reformat) package sources (exclude vendor dir if existed).

  # 静态代码检查
  lint               Check syntax and styling of go sources.

  # 测试类命令
  test               Run unit test.
  cover              Run unit test and get test coverage.

  # 构建类命令
  build              Build source code for host platform.
  build.multiarch    Build source code for multiple platforms. See option PLATFORMS.

  # Docker镜像打包类命令
  image              Build docker images for host arch.
  image.multiarch    Build docker images for multiple platforms. See option PLATFORMS.
  push               Build docker images for host arch and push images to registry.
  push.multiarch     Build docker images for multiple platforms and push images to registry.

  # 部署类命令
  deploy             Deploy updated components to development env.

  # 清理类命令
  clean              Remove all files that are created by building.

  # 其他命令，不同项目会有区别
  release            Release project
  verify-copyright   Verify the boilerplate headers for all files.
  ca                 Generate CA files for all project components.
  install            Install project system with all its components.
  swagger            Generate swagger document.
  tools              install dependent tools.

  # 帮助命令
  help               Show this help info.

# 选项
Options:
  DEBUG        Whether to generate debug symbols. Default is 0.
  BINS         The binaries to build. Default is all of cmd.
               This option is available when using: make build/build.multiarch
               Example: make build BINS="iam-apiserver iam-authz-server"
  ...
```

通常而言，Go 项目的 Makefile 应该实现以下功能：**格式化代码**、**静态代码检查**、**单元测试**、**代码构建**、**文件清理**、**帮助**等等。如果通过 docker 部署，还需要有 **docker镜像打包**功能。因为 Go 是跨平台的语言，所以构建和 docker 打包命令，还要能够支持不同的 CPU 架构和平台。为了能够更好地控制 Makefile 命令的行为，还需要支持 Options。

为了方便查看 Makefile 集成了哪些功能，我们需要支持 help 命令。help 命令最好通过解析 Makefile 文件来输出集成的功能，例如：

``` makefile
## help: Show this help info.
.PHONY: help
help: Makefile
  @echo -e "\nUsage: make <TARGETS> <OPTIONS> ...\n\nTargets:"
  @sed -n 's/^##//p' $< | column -t -s ':' | sed -e 's/^/ /'
  @echo "$$USAGE_OPTIONS"
```


# 设计合理的 Makefile 结构

对于大型项目来说，需要管理的内容很多，所有管理功能都集成在一个 Makefile 中，可能会导致 Makefile 很大，难以阅读和维护，所以建议采用分层的设计方法，根目录下的 Makefile 聚合所有的 Makefile 命令，具体实现则按功能分类，放在另外的 Makefile 中。

<img src="https://img1.kiosk007.top/static/images/makefile/makefile_struct.webp">

在上面的 Makefile 组织方式中，根目录下的 Makefile 聚合了项目所有的管理功能，这些管理功能通过 Makefile 伪目标的方式实现。同时，还将这些伪目标进行分类，把相同类别的伪目标放在同一个 Makefile 中，这样可以使得 Makefile 更容易维护。对于复杂的命令，则编写成独立的 shell 脚本，并在 Makefile 命令中调用这些 shell 脚本。

``` bash

├── Makefile
├── scripts
│   ├── gendoc.sh
│   ├── make-rules
│   │   ├── gen.mk
│   │   ├── golang.mk
│   │   ├── image.mk
│   │   └── ...
    └── ...
```

为了跟 Makefile 的层级相匹配，golang.mk 中的所有目标都按go.xxx这种方式命名。通过这种命名方式，我们可以很容易分辨出某个目标完成什么功能，放在什么文件里，这在复杂的 Makefile 中尤其有用。例如


``` makefile

include scripts/make-rules/golang.mk
include scripts/make-rules/image.mk
include scripts/make-rules/gen.mk
include scripts/make-rules/...

## build: Build source code for host platform.
.PHONY: build
build:
  @$(MAKE) go.build

## build.multiarch: Build source code for multiple platforms. See option PLATFORMS.
.PHONY: build.multiarch
build.multiarch:
  @$(MAKE) go.build.multiarch

## image: Build docker images for host arch.
.PHONY: image
image:
  @$(MAKE) image.build

## push: Build docker images for host arch and push images to registry.
.PHONY: push
push:
  @$(MAKE) image.push

## ca: Generate CA files for all iam components.
.PHONY: ca
ca:
  @$(MAKE) gen.ca
```

# 掌握 Makefile 编写技巧

## 技巧 1：善用通配符和自动变量

Makefile 允许对目标进行类似正则运算的匹配，主要用到的通配符是%。通过使用通配符，可以使不同的目标使用相同的规则，从而使 Makefile 扩展性更强，也更简洁。

这里，我们来看一个具体的例子，tools.verify.%（位于[scripts/make-rules/tools.mk](https://github.com/marmotedu/iam/blob/master/scripts/make-rules/tools.mk#L17)文件中）定义如下：

``` makefile

TOOLS ?=$(BLOCKER_TOOLS) $(CRITICAL_TOOLS) $(TRIVIAL_TOOLS)

.PHONY: tools.install
tools.install: $(addprefix tools.install., $(TOOLS))

.PHONY: tools.install.%
tools.install.%:
	@echo "===========> Installing $*"
	@$(MAKE) install.$*

tools.verify.%:
	@if ! which $* &>/dev/null; then $(MAKE) tools.install.$*; fi

```

`make tools.verify.swagger`, `make tools.verify.mockgen`等均可以使用上面定义的规则，%分别代表了`swagger`和`mockgen`。

如果执行了 `make install` , 则执行的逻辑是，将执行`$(addprefix tools.install., $(TOOLS))` 这个命令的意思这时执行 `tools.install`,进一步拼装了一系列的`tools.install.xxxxxx` 并执行。



如果不使用`%`，则我们需要分别为`tools.verify.swagger`和`tools.verify.mockgen`定义规则，很麻烦，后面修改也困难。

另外，这里也能看出`tools.verify.%`这种命名方式的好处：tools 说明依赖的定义位于scripts/make-rules/tools.mk Makefile 中；verify说明tools.verify.%伪目标属于 verify 分类，主要用来验证工具是否安装。通过这种命名方式，你可以很容易地知道目标位于哪个 Makefile 文件中，以及想要完成的功能。另外，上面的定义中还用到了自动变量`$*`，用来指代被匹配的值`swagger、mockgen`。


## 技巧 2：善用函数

Makefile 自带的函数能够帮助我们实现很多强大的功能。所以，在我们编写 Makefile 的过程中，如果有功能需求，可以优先使用这些函数。我把常用的函数以及它们实现的功能整理在了 [Makefile 常用函数列表](https://github.com/marmotedu/geekbang-go/blob/master/makefile/Makefile%E5%B8%B8%E7%94%A8%E5%87%BD%E6%95%B0%E5%88%97%E8%A1%A8.md) 中


## 技巧 3：依赖需要用到的工具

如果 Makefile 某个目标的命令中用到了某个工具，可以将该工具放在目标的依赖中。这样，当执行该目标时，就可以指定检查系统是否安装该工具，如果没有安装则自动安装，从而实现更高程度的自动化。例如，/Makefile 文件中，format 伪目标，定义如下：


``` makefile

.PHONY: format
format: tools.verify.golines tools.verify.goimports
  @echo "===========> Formating codes"
  @$(FIND) -type f -name '*.go' | $(XARGS) gofmt -s -w
  @$(FIND) -type f -name '*.go' | $(XARGS) goimports -w -local $(ROOT_PACKAGE)
  @$(FIND) -type f -name '*.go' | $(XARGS) golines -w --max-len=120 --reformat-tags --shorten-comments --ignore-generated .
```

可以看到，format 依赖tools.verify.golines tools.verify.goimports。我们再来看下tools.verify.golines的定义：

``` makefile

tools.verify.%:
  @if ! which $* &>/dev/null; then $(MAKE) tools.install.$*; fi
```
通过`tools.verify.%`规则定义，我们可以知道，`tools.verify.%`会先检查工具是否安装，如果没有安装，就会执行`tools.install.$*`来安装。如此一来，当我们执行`tools.verify.%`目标时，如果系统没有安装 golines 命令，就会自动调用go get安装，提高了 Makefile 的自动化程度。





## 编写可扩展的 Makefile

举个例子，执行 `make go.build` 时能够构建 cmd/ 目录下的所有组件，也就是说，当有新组件添加时， `make go.build` 仍然能够构建新增的组件。

``` makefile

COMMANDS ?= $(filter-out %.md, $(wildcard ${ROOT_DIR}/cmd/*))
BINS ?= $(foreach cmd,${COMMANDS},$(notdir ${cmd}))

.PHONY: go.build
go.build: go.build.verify $(addprefix go.build., $(addprefix $(PLATFORM)., $(BINS)))
.PHONY: go.build.%               

go.build.%:             
  $(eval COMMAND := $(word 2,$(subst ., ,$*)))
  $(eval PLATFORM := $(word 1,$(subst ., ,$*)))
  $(eval OS := $(word 1,$(subst _, ,$(PLATFORM))))           
  $(eval ARCH := $(word 2,$(subst _, ,$(PLATFORM))))                         
  @echo "===========> Building binary $(COMMAND) $(VERSION) for $(OS) $(ARCH)"
  @mkdir -p $(OUTPUT_DIR)/platforms/$(OS)/$(ARCH)
  @CGO_ENABLED=0 GOOS=$(OS) GOARCH=$(ARCH) $(GO) build $(GO_BUILD_FLAGS) -o $(OUTPUT_DIR)/platforms/$(OS)/$(ARCH)/$(COMMAND)$(GO_OUT_EXT) $(ROOT_PACKAGE)/cmd/$(COMMAND)
```

当执行make go.build 时，会执行 go.build 的依赖 `$(addprefix go.build., $(addprefix $(PLATFORM)., $(BINS)))` , `addprefix`函数最终返回字符串 `go.build.linux_amd64.iamctl go.build.linux_amd64.iam-authz-server go.build.linux_amd64.iam-apiserver ... `，这时候就会执行 `go.build.%` 伪目标。在 `go.build.% `伪目标中，通过 `eval`、`word`、`subst` 函数组合，算出了 COMMAND 的值 `iamctl/iam-apiserver/iam-authz-server/...`，最终通过 `$(ROOT_PACKAGE)/cmd/$(COMMAND)` 定位到需要构建的组件的 main 函数所在目录。


通过以下方法可以获取到 cmd 下所有的组件名。

``` bash
COMMANDS ?= $(filter-out %.md, $(wildcard ${ROOT_DIR}/cmd/*))
BINS ?= $(foreach cmd,${COMMANDS},$(notdir ${cmd}))
```

接着，通过使用通配符和自动变量，自动匹配到go.build.linux_amd64.iam-authz-server 这类伪目标并构建。
