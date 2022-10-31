# Go Module 基本使用

Go 1.13版本之后新的包管理器Modules趋于成熟，目前越来越多的开源项目已经支持Go Modules，典型的如etcd。Go具有相当长的包管理工具变迁史，各种包管理工具层出不穷，究其原因，还是官方没有实现足够好用包管理工具。本文不对部分基础知识做详解，主要重点是Go Modules

参考：https://roberto.selbach.ca/intro-to-go-modules/

<!--more-->

# 背景
几乎所有的包管理工具在Go 1.11版本之前都绕不开GOPATH这个环境变量。GOPATH主要用来放置项目依赖包的源代码，GOPATH不区分项目，代码中任何import的路径均从GOPATH为根目录开始；但现在GOPATH已经不够用了。
# 概念介绍
## 开启 go module
`go module` 没有默认开启，开关通过系统环境变量 `GO111MODULE` 控制。 
这个变量有三个可选值 

> 1.16 版本起，环境变量 GO111MODULE 的默认值正式修改为 on。

这个变量有三个可选值 
- **auto**: 默认值，如果项目在 `$GOPATH` 目录内则使用传统的 GOPATH 方式管理，否则使用 Go module 方式管理。 go1.13 后只要项目有 go.mod 文件，即使在 GOPATH 里，也会使用 Go module 方式管理。
- **on** : 不论项目在 `$GOPATH` 之内还是之外，全部使用 Go module 方式管理。 
- **off** : 项目在 `$GOPATH` 之内使用传统方式管理，不支持项目在 `$GOPATH` 之外。


## 模块 module

go module 模式引入了一个新的概念 - 模块(module)，一个模块是一组包(package)的集合，这些包共用同一个版本信息。 
模块里记录了准确的依赖信息。 
一个包含 go.mod 的目录及其子目录构成了一个模块 (module)；一个目录即是一个包 (package)；再加上代码托管平台 (github, gitlab) 的仓库 (repositories)，他们的关系如下： 
- 一个仓库包含一个或者多个模块 
- 一个模块包含一个或多个包 
- 一个包包含该目录下的所有 go 源码文件 


## go.mod, go.sum

使用 go module 模式进行依赖管理会生成 go.mod, go.sum 两个文件。 
go.mod 有四个指令 `module`, `require`, `replace`, `exclude`。 
常见的 go.mod 文件如下 

``` golang
module github.com/lucas-clemente/quic-go

go 1.15

require (
	github.com/cheekybits/genny v1.0.0
	github.com/francoispqt/gojay v1.2.13
	github.com/golang/mock v1.6.0
	github.com/marten-seemann/qpack v0.2.1
	github.com/marten-seemann/qtls-go1-15 v0.1.5
	github.com/marten-seemann/qtls-go1-16 v0.1.4
	github.com/marten-seemann/qtls-go1-17 v0.1.0-rc.1
	github.com/onsi/ginkgo v1.16.4
	github.com/onsi/gomega v1.13.0
	golang.org/x/crypto v0.0.0-20200622213623-75b288015ac9
	golang.org/x/net v0.0.0-20210428140749-89ef3d95e781
	golang.org/x/sync v0.0.0-20210220032951-036812b2e83c
	golang.org/x/sys v0.0.0-20210510120138-977fb7262007
	gopkg.in/check.v1 v1.0.0-20180628173108-788fd7840127 // indirect
)

replace (
    golang.org/x/crypto v0.0.0-20200622213623-75b288015ac9 => github.com/google/x/crypto v0.0.0-20200622213623-75b288015ac9
)

```

module 用来定义当前模块名称 
- **require**: 是描述依赖信息 
- **replace** 用于是用指定依赖（后者）去代替前者。比如 github 的仓库项目替代 golang.org 的。 
- **exclude** 用来忽略某些特定版本


> Go1.16 针对 Go modules 放出了一个新特性，能够让维护 Go mod 库的开发者拥有反复吃 “后悔药” 的权力，提醒开发者已发布的脏版本存在问题。`go mod retract`
参考：[Go1.16 新特性 go mod retract](https://eddycjy.com/posts/go/go1.16-mod/)

## 版本

go module 的版本通过仓库的 tag 来确定，所以推荐 master 上的重要版本都打上 tag。

对于没有打 tag 的仓库，go.mod 就会很丑陋，它的格式是 v0.0.0-<time>-<commit id> 。它的含义是v0.0.0-yyyymmddhhmmss-abcdefabcdef。


# 使用

## 初始化项目

对于一个新的项目，需要执行` go mod init xxxx `初始化 `go.mod` 文件。xxxx 为模块路径 (比如 `github.com/pkg/test`)，即 import 使用的路径。
然后执行 `go mod tidy` 整理依赖，这个命令会检索项目添加 `go.mod` 里面没有的依赖，同时也会删除不再使用的依赖。 

## 添加新依赖

1. 自动添加

在代码里面 import 了某个依赖后，执行 `go build`、`go run`、`go test` 就会自动添加该依赖的最新版到 go.mod。

2. 手动添加

如果新增了一个依赖，可以使用 

```
go get github.com/pkg/xxx@master
```

@master 这里也可以指定特定版本，比如 @v1.0.1，或者 @latest 更新到最新版本。也可以指定特定的 commit id，比如 @aabbccdd。

## go mod 命令

``` bash
go mod
The commands are:
  download    download modules to local cache (下载依赖的module到本地cache))
  edit        edit go.mod from tools or scripts (编辑go.mod文件)
  graph       print module requirement graph (打印模块依赖图))
  init        initialize new module in current directory (再当前文件夹下初始化一个新的module, 创建go.mod文件))
  tidy        add missing and remove unused modules (增加丢失的module，去掉未用的module)
  vendor      make vendored copy of dependencies (将依赖复制到vendor下)
  verify      verify dependencies have expected content (校验依赖)
  why         explain why packages or modules are needed (解释为什么需要依赖)
```

## 总结

初始化项目使用 `go mod init xxxx`
日常使用最多的命令就是 `go get -u xxxx` 来添加依赖和 `go mod tidy` 来清理依赖。


# 版本号

-  release version（发布版本）

发布版本是一个由点号组成的三个正整数。比如 v1.2.3，从左到位每一位整数分别被称为 major, minor, patch。

- pseudo version（伪版本）

伪版本号是通过编码一个版本唯一标识（比如 git 的 commit id)和一个时间戳组成的，比如 v0.0.0-20191109021931-daa7c04131f5。主要在两个场景使用，一是对于不是 go module 的仓库，二是一些场景下获取不到合适的 tag 信息，比如指定分支。

- pre-release version(预发布版本）

预发布版本是发布版本后接一个预发布后缀。预发布后缀是 - 接一个字符串或者 + 接一个 build 元信息。比如 v8.0.5-pre, v2.0.9+meta 
带有预发布后缀的版本都为被认为为不稳定版本，这类版本会认为与其他版本不兼容。



# Go 环境

``` bash
// 指定不走 proxy 的包路径，用于公司内网的仓库不走代理
export GOPRIVATE="*.taobao.com,*.abc.com"
// SUMDB 的代理，SUMDB 用于校验版本是否被恶意修改
export GOSUMDB="sum.golang.google.cn"

// GOPROXY，拉取依赖库使用的代理。
// 适用于 GO 版本 1.15
export GOPROXY="https://goproxy.cn|direct"
```
