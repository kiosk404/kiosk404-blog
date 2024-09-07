---
title: GoLang 静态代码分析工具
author: kiosk
tags:
  - golang
categories:
  - programming
date: 2021-12-18 22:01:00
---
在日常Golang编程过程中，需要对 Go 代码做静态代码检查。虽然 Go 命令提供了 `go vet` 和 `go tool vet`，但是它们检查的内容还不够全面。go 的 vet 工具可以用来检查 go 代码中可以通过编译但仍然有可能存在错误的代码。包括并发访问安全、死锁、泄露上下文、结构体反射等等，参考[文章](https://zhuanlan.zhihu.com/p/357406395)


<!--more-->

可用于 Go 语言代码分析的工具有很多，比如 golint、gofmt、misspell 等，如果一一引用配置，就会比较烦琐，所以通常我们不会单独地使用它们，而是使用 golangci-lint。

golangcli-lint 官网 : https://golangci-lint.run/


golangci-lint 是一个集成工具，它集成了很多静态代码分析工具，便于我们使用。通过配置这一工具，我们可以很灵活地启用需要的代码规范检查。


# 安装

`golangci-lint` 本身是通过 GO 编写，所以可以直接使用 `go get` 安装。

``` bash
go get github.com/golangci/golangci-lint/cmd/golangci-lint
```

或者按照官网的方式[安装](https://golangci-lint.run/usage/install/)

``` bash
# binary will be $(go env GOPATH)/bin/golangci-lint
curl -sSfL https://raw.githubusercontent.com/golangci/golangci-lint/master/install.sh | sh -s -- -b $(go env GOPATH)/bin v1.43.0

# or install it into ./bin/
curl -sSfL https://raw.githubusercontent.com/golangci/golangci-lint/master/install.sh | sh -s v1.43.0

# In alpine linux (as it does not come with curl by default)
wget -O- -nv https://raw.githubusercontent.com/golangci/golangci-lint/master/install.sh | sh -s v1.43.0

```

安装完成后，在终端输入如下命令，检测是否安装成功

``` bash
golangci-lint --version
golangci-lint has version v1.43.0 built 
```


# 优势

选择 `golangci-lint`，是因为它具有其他静态代码检查工具不具备的一些优点。

- 速度非常快：golangci-lint 是基于 gometalinter 开发的，但是平均速度要比 gometalinter 快 5 倍。golangci-lint 速度快的原因有三个：可以并行检查代码；可以复用 go build 缓存；会缓存分析结果。
- 可配置：支持 YAML 格式的配置文件，让检查更灵活，更可控。
- IDE 集成：可以集成进多个主流的 IDE，例如 VS Code、GNU Emacs、Sublime Text、Goland 等。
- linter 聚合器：1.41.1 版本的 golangci-lint 集成了 76 个 linter，不需要再单独安装这 76 个 linter。并且 golangci-lint 还支持自定义 linter。
- 最小的误报数：golangci-lint 调整了所集成 linter 的默认设置，大幅度减少了误报。
- 良好的输出：输出的结果带有颜色、代码行号和 linter 标识，易于查看和定位。

`golangci-lint` 有诸多大公司背书，，例如 Google、Facebook、Istio、Red Hat OpenShift 等。




# 运行
安装成功 golangci-lint 后，就可以使用它进行代码检查了，示例代码

``` golang 
package main

import "os"

func main() {
	os.Open("./main.go")
}

```


如下, 提示对返回内容进行错误检查。

``` bash
golangci-lint run 
main.go:11:9: Error return value of `os.Open` is not checked (errcheck)
        os.Open("./main.go")
               ^

```


# golangci-lint 命令

我们可以通过执行 golangci-lint -h 查看其用法，golangci-lint 支持的子命令见下表：

|子命令|功能|
|:--:|:--:|
|cache| 缓存控制，并打印缓存信息|
|completion|输出 bash\fish\powershell\zsh 补全脚本|
|config|打印golangci-lint使用的配置文件路径|
|help|打印帮助信息|
|linters|打印golangci-lint所支持的全部linter|
|run|使用golangci-lint对代码进行检查|
|version|打印golangci-lint的版本号|

- **run 命令**: run 命令执行 golangci-lint，对代码进行检查，是 golangci-lint 最为核心的一个命令。run 没有子命令，但有很多选项。

- **cache 命令**: cache 命令用来进行缓存控制，并打印缓存的信息。它包含两个子命令, `status` 、 `clean`

``` bash
golangci-lint cache status
Dir: /home/ubuntu/.cache/golangci-lint
Size: 71.3KiB

```

- **completion 命令**: completion 命令包含 4 个子命令 bash、fish、powershell 和 zsh，分别用来输出 bash、fish、powershell 和 zsh 的自动补全脚本。

``` bash

$ golangci-lint completion bash > ~/.golangci-lint.bash
$ echo "source '$HOME/.golangci-lint.bash'" >> ~/.bashrc
$ source ~/.bashrc
```

- **linters 命令**: linters 命令可以打印出 golangci-lint 所支持的 linter，并将这些 linter 分成两类，分别是配置为启用的 linter 和配置为禁用的 linter，例如：

``` bash

$ golangci-lint linters
Enabled by your configuration linters:
...
deadcode: Finds unused code [fast: true, auto-fix: false]
...
Disabled by your configuration linters:
exportloopref: checks for pointers to enclosing loop variables [fast: true, auto-fix: false]
...
```

运行指定的 linter


你可以传入参数-E/--enable来使某个 linter 可用，也可以使用-D/--disable参数来使某个 linter 不可用。下面的示例仅仅启用了 errcheck linter：

``` bash
$ golangci-lint run --no-config --disable-all -E errcheck ./...
```


# golangci-lint 配置

`golangci-lint` 的配置比较灵活，比如你可以自定义要启用哪些 `linter`。`golangci-lint` 默认启用的 `linter`，包括这些：

- deadcode - 死代码检查
- errcheck - 返回错误是否使用检查
- gosimple - 检查代码是否可以简化
- govet - 代码可疑检查，比如格式化字符串和类型不一致
- ineffassign - 检查是否有未使用的代码
- staticcheck - 静态分析检查
- structcheck - 查找未使用的结构体字段
- typecheck - 类型检查
- unused - 未使用代码检查
- varcheck - 未使用的全局变量和常量检查



可以使用 `golangci-lint linters` 查看;

``` bash
golangci-lint linters
Enabled by your configuration linters:
deadcode: Finds unused code [fast: false, auto-fix: false]
errcheck: Errcheck is a program for checking for unchecked errors in go programs. These unchecked errors can be critical bugs in some cases [fast: false, auto-fix: false]
...

Disabled by your configuration linters:
asciicheck: Simple linter to check that your code does not contain non-ASCII identifiers [fast: true, auto-fix: false]
bidichk: Checks for dangerous unicode character sequences [fast: true, auto-fix: false]
bodyclose: checks whether HTTP response body is closed successfully [fast: false, auto-fix: false]
contextcheck: check the function whether use a non-inherited context [fast: false, auto-fix: false]
cyclop: checks function and package cyclomatic complexity [fast: false, auto-fix: false]
...

```

这些 linter 分为两类，一类是默认启用的，另一类是默认禁用的。每个 linter 都有两个属性：

- fast：true/false，如果为 true，说明该 linter 可以缓存类型信息，支持快速检查。因为第一次缓存了这些信息，所以后续的运行会非常快。

- auto-fix：true/false，如果为 true 说明该 linter 支持自动修复发现的错误；如果为 false 说明不支持自动修复。


如果要修改默认启用的 `linter`，就需要对 `golangci-lint` 进行配置。即在项目根目录下新建一个名字为 `.golangci.yml` 的文件，这就是 `golangci-lint` 的配置文件。在运行代码规范检查的时候，`golangci-lint` 会自动使用它。假设我只启用 unused 检查，可以这样配置

``` yaml
linters:
  disable-all: true
  enable:
    - unused

```

更详细的配置内容，你可以参考 [Configuration](https://golangci-lint.run/usage/configuration/)
[IAM 项目参考](https://github.com/marmotedu/iam/blob/master/.golangci.yaml)

``` yaml

     linters-settings:
  golint:
    min-confidence: 0
  misspell:
    locale: US
linters:
  disable-all: true
  enable:
    - typecheck
    - goimports
    - misspell
    - govet
    - golint
    - ineffassign
    - gosimple
    - deadcode
    - structcheck
    - unused
    - errcheck
service:
  golangci-lint-version: 1.32.2 # use the fixed version to not introduce new linters unexpectedly

```

# 忽略检查

- 某一行忽略检查

``` golang
const a = 1 //nolint
```

- 某个函数块忽略检查

代码中忽略检查, 注释中加 //nolint
``` golang

//nolint
func allIssuesInThisFunctionAreExcluded() *string {
  // ...
}

//nolint:govet
var (
  a int
  b int
)
```

- 文件忽略检查

在 package xx 上面一行添加//nolint注释。
``` golang
//nolint:unparam
package pkg
...
```


# 继承到CI


集成到 `CI` 代码检查一定要集成到 `CI` 流程中，效果才会更好，这样开发者提交代码的时候，`CI` 就会自动检查代码，及时发现问题并进行修正。

不管你是使用 `Jenkins`，还是 `Gitlab CI`，或者 `Github Action`，都可以通过 `Makefile` 的方式运行 `golangci-lint` 。现在我在项目根目录下创建一个 `Makefile` 文件，并添加如下代码：

``` 
getdeps:
   @mkdir -p ${GOPATH}/bin
   @which golangci-lint 1>/dev/null || (echo "Installing golangci-lint" && go get github.com/golangci/golangci-lint/cmd/golangci-lint@v1.32.2)
lint:
   @echo "Running $@ check"
   @GO111MODULE=on ${GOPATH}/bin/golangci-lint cache clean
   @GO111MODULE=on ${GOPATH}/bin/golangci-lint run --timeout=5m --config ./.golangci.yml
verifiers: getdeps lint

```

