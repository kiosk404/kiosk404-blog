---
title: 再见Hexo，你好Hugo
author: kiosk
tags: ["hugo"]
categories:
  - blog
date: 2022-04-05 11:23:00
---
最近两年一直在用 hexo 来写博客，第一个认识的 hexo 静态托管类网站记得还是小米运维部的网站 noops.me ，但是时光荏苒，小米的技术博客16年就停更了。hexo 的生态也比较完整，也有很多不错的博客主题，但是这次想着和 hexo 说再见了。


<!--more-->

# 再看一眼 Hexo

<img src="https://img1.kiosk007.top/static/images/bye_hexo/kiosk007.png">

本次一时兴起想要迁移到 Hugo 主要有以下几个原因吧
- **作为一个 Gopher ，怎能在NodeJS这寄人篱下！！！**
- Hexo 的其实也换了不少主题了，但实际上没有一个是特别喜欢的。
- Hugo 既可以简洁也可以花里胡哨
- 实在是不喜欢也不熟悉 NodeJS


# Hugo

引用一下Hugo官网的描述

>The world’s fastest framework for building websites.
>Hugo是一个非常受欢迎的、开源的静态网站生成工具，和Hexo类似。 它速度快，扩展性强。

更多的关于Hugo的介绍，请参考Hugo的官网 https://gohugo.io/ 。Hugo 拥有众多优势，当然也有一些缺点。hugo 支持 [emoji](https://emojis.github.io/) 🤪, Hugo使用Go语言，生成速度更快。但是也有缺点，很多主题默认支持的功能不够全面，不过好在都是可插件拓展的，可以自己随时补充。由于可以看Golang 的源码，所以可以拓展起来，也比hexo好理解。


# 开始迁移

首先是安装 hugo, 因为我是 ubuntu 系统，所以可以直接使用 apt-get。详细的步骤其实参考[官网](https://gohugo.io/getting-started/quick-start/)就好。

## 文章链接

对于个人博客的迁移而言，保证文章链接不变是最重要的。在 hexo 中，文件名与链接的对应关系如下：

``` yaml
permalink: blog/:year/:month/:day/:title/
```
对应的 markdown 文件是

``` bash
source/_posts/:year-:month-:day-:title.markdown
```
这样的好处是文件按照时间排列，管理方便。

Hugo 则不同，它默认会用文件中 frontmatter 的 date，而非文件名中的。参考[这篇博客](https://liujiacai.net/blog/2020/12/05/hexo-to-hugo/)最终在 hugo 添加下面的配置完美解决：

``` toml
[frontmatter]
  date = [":filename", ":default", ":fileModTime"]

[permalinks]
  post = "/blog/:year/:month/:day/:slug/"
```

这样 Hugo 就会首先从文件名中解析日期，同时也会解析 slug；失败时再从 frontmatter 中解析，更多用法可参考官方文档 Configure Dates。

同时，为了避免 `hugo new post/year-month-day-slug.org` 时标题中带日期，修改[默认模板](https://gohugo.io/content-management/archetypes/)如下：

```
title: {{ replace  .Name "-" " " | replaceRE "^\\d{4} \\d{2} \\d{2} (.*)" "$1" | title }}
```

> 不过最终还是觉得 hexo 的文章链接模式太丑，换了回来。


## Frontmatter

[Frontmatter](https://gohugo.io/content-management/front-matter) 定义了每篇文章的属性，比如标题、分类等。这也是在 hexo 迁移到 hugo 时问题最多的地方，根本原因在于 hexo 对 frontmatter 格式较宽松，而 hugo 则比较严格。

下面一个 hugo 中标准的 frontmatter（除 yaml 外，还可以是 toml/json）：

``` yaml
categories:
- Development
- VIM
date: "2012-04-06"
description: spf13-vim is a cross platform distribution of vim plugins and resources
  for Vim.
slug: spf13-vim-3-0-release-and-new-website
tags:
- .vimrc
- plugins
- spf13-vim
- vim
title: spf13-vim 3.0 release and new website
```


这里的问题是，hexo admin 生成的文章，只有下面的 `-----` ，而在hugo里要求上下都需要有 `-----` 包围着。


# 主题选择

开始比较中意的是 [hugo-theme-console](https://github.com/mrmierzejewski/hugo-theme-console/)。这款主题采用 Linux 终端风格。非常适合极客程序员的风格。

<img src="https://img1.kiosk007.top/static/images/bye_hexo/preview.png">
但是其采用的 [terminal css](https://terminalcss.xyz/) 搭配 markdown 着实bug不少，主题的issue也提出了几个，但是看情况其作者也是无能为力，毕竟是引用，对于本来前端就不太熟的我，还是放弃了。

在摸索 hugo 主题的时候，着实有想法自己重写一套主题，虽然没能最终实现，但是也学会不少关于 hugo 开发的一些事情。这里还是比较有意思的，值得记录一下。


## 写一个主题

写一个 Hugo 的主题其实并不复杂，比我想象中的要容易得多（之前总是被它繁杂的文档望而却步），当然也得益于 Hugo 这个项目日趋成熟，很多相同的部分、功能，已经内置，在主题模板中只需稍加引入即可。

首先，Hugo 的命令行工具提供了一个命令来生成主题的脚手架文件。

``` bash
hugo new theme futu
```

有了这个脚手架之后，首先也是最关键的入口文件是 `layouts/_default/baseof.html`，在这个文件里可以定义网站的基本组成部分，比如 head，main，footer 等等，下面是我主题里这个文件的内容。

``` html
<!DOCTYPE html>
<html>
  <head>
    {{- partial "head.html" . -}}
    <title>{{ block "title" . }}{{ .Site.Title }}{{ end }}</title>
  </head>
  <body>
    {{- partial "header.html" . -}}
    <main class="container">{{- block "main" . }} {{- end }}</main>
    {{- partial "footer.html" . -}} {{- partial "script.html" . -}}
  </body>
</html>
```

想必应该很好读懂，`{{- partial "head.html" . -}}`，两个大括号是 Hugo 的模板语言标记，在里面可以定义变量，调用函数等，这里的 `partial` 函数会引用 `head.html` 的内容，并将当前上下文 . 传入，也许你也留意到了内侧括号旁的横线-，那个是用来清除模板周围的空格符号，比如左边的-意味着将左侧模板左侧的空白符号统统清除，详情看[这里](https://gohugo.io/templates/introduction/#whitespace)

## 文档阅读

hugo 的文档是算比较全面的了，对于主题开发来讲，文档大体分为三大块，分别是

- [Templates](https://gohugo.io/templates/)
- [Functions](https://gohugo.io/functions/)
- [Variables](https://gohugo.io/variables/)

**模板**

可以看到主题的 `layout` 目录下有很多文件和目录。简单来说，对于`主页`，`文章页`，`归档页`，`分类列表页`，`分类页`这些不同种类的页面，都有相应的地方读取模板，如果某个页面有多个地方的模板文件相匹配，则只有优先级高的模板会被使用。其对照关系如下：

|  __页面__      |  __模板地址__          | __说明__     |
| -------------|:---------------:| :-----------------|
|主页	          |    layouts/index.html	      |  |
|文章页 (posts 目录下)| layouts/posts/single.html   | 在 /posts/ 目录下的普通文章|
|   其他文章页   | layouts/_default/single.html| 如 /about，由 about.md 生成     |
|   归档页	     | layouts/_default/section.html	 |  如 /posts/ |
|   分类列表页	| layouts/_default/terms.html	    | 如 /tags/ |
|   分类页	|  layouts/_default/term.html	|如 /tags/中文/ |

除此之外，hugo 还内置了不少通用的模板，称为 [Internal Templates](https://gohugo.io/templates/internal) ，比如 disqus，google analytics，可以通过 template 函数引入，比如在合适的地方 `{{ template "_internal/disqus.html" . }}`，这样在配置文件中定义了 disqusShortname 之后，就可以显示评论了。

**函数**

在 hugo 的模板里经常可以看到一些 `range` 函数，其就是 hugo 定义的一批函数。比如 `{{ with .Site.GetPage "/blog/my-post.md" }}{{ .Title }}{{ end }}` 中的`.Site.GetPage`就是获取一篇位于 `/blog/my-post.md` 的这篇文章的位置。

## 搜索
考虑再三，期初决定使用 `even` 这个模板，但是这个模板默认也没有搜索功能，于是还需要自己手动添加相关的功能。
参考网上的一些文章，可以利用`algolia` 实现站点内的文章搜索。[hugo添加algolia搜索支持](https://edward852.github.io/post/hugo%E6%B7%BB%E5%8A%A0algolia%E6%90%9C%E7%B4%A2%E6%94%AF%E6%8C%81/#%E7%94%9F%E6%88%90%E7%B4%A2%E5%BC%95%E6%96%87%E4%BB%B6)。 这篇文章讲得挺全的，我看网上的解决方案基本也是参考这篇文章的。

不过后来换了 `loveit` 这个主题，由于主题默认支持了搜索，而且搜索做的很棒，我也就默认使用了这个。

按照上面的博客很快就可以搞定搜索功能，但是主要就是麻烦再自动化这里了。这个先简单介绍一下，详细的在部署那里说。自动上传索引这里很多文章都介绍了 `atomic-algolia` 这个工具。

- 安装 atomic-algolia 包

``` bash
npm init  // 不懂的就回车好了
npm install atomic-algolia --save
```
- 修改目录下的 package.json 文件，在 scripts 下添加 "algolia": "atomic-algolia"

``` json
"scripts": {
    "test": "echo \"Error: no test specified\" && exit 1",
    "algolia": "atomic-algolia"
},
```
- 项目根目录下新建 .env 文件，内容如下：

```
ALGOLIA_APP_ID=你的Application ID
ALGOLIA_INDEX_NAME=你的索引名字
ALGOLIA_INDEX_FILE=public/algolia.json
ALGOLIA_ADMIN_KEY=你的Admin API Key

```

另外特别注意 `ALGOLIA_ADMIN_KEY` 可以用来管理你的索引，所以尽量不要提交到公共仓库。
一般的做法是在你的自动化部署工具中设置环境变量。

所以最后我们看看如何利用 CI 去跑把。


## 评论系统

`loveit` 自带了很多评论系统足够满足满足大多数人的需求，我选了主题默认的评论系统 [Valine](https://github.com/xCss/Valine)。
这个评论系统也是需要注册的。具体过程参考 [这篇博客](https://huangzhongde.cn/post/2020-02-20-hugo-comments-plugin-valine/) 吧。也写得算挺详细的了。




# 部署

Hugo官方提供了多种部署方式，其中主要包含 [Host on GitHub](https://gohugo.io/hosting-and-deployment/hosting-on-github/) 和 [Deployment with Rsync](https://gohugo.io/hosting-and-deployment/deployment-with-rsync/) 结合的方式。

具体实现是直接放弃自己远程主机上的仓库，并使用 GitHub Actions 进行站点的部署。


## GitHub Actions

[GitHub Actions](https://docs.github.com/cn/actions/quickstart) 是 GitHub 官方提供的一种自动化、定制化的工作流，包括了 CI/CD。关于 CI/CD 的简单理解：

- CI：持续集成（Continuous Integration），使用 Git 向代码仓库推送代码后，后台将会自动进行构建、测试等工作。
- CD：持续交付（Continuous Delivery），推送完成的代码（经过了自动构建、测试等流程），最终部署到生产服务器，供客户直接使用。

部署 Hugo 博客的步骤为在提交 git 记录时，触发 github actions 将 public 目录下的内容同步至我们的服务器Nginx相关目录。

关于这个的介绍，可以参考[阮一峰的博客](https://www.ruanyifeng.com/blog/2019/09/getting-started-with-github-actions.html)
GitHub Actions 的具体实现是在当前工作目录下创建 .github/workflows/，并在目录中添加 .yml 脚本文件，每个 .yml 文件都代表了一个 workflow。目录结构如下：
``` bash
- .github
  |- workflows
    |- workflow1.yml
    |- workflow2.yml
```

## 准备工作
- 远程主机配置

部署流程使用 rsync 进行文件的同步工作， rsync 默认是基于 SSH，需要提前在自己的远程主机上安装 rsync，并准备密钥对。

``` bash
# 创建一个用户 
useradd -c "worker" -m work
su - work
cd .ssh
# 创建秘钥
ssh-keygen -t ed25519 -C "xxxx@xxxx.com"
...
cat id_ed25519.pub >> authorized_keys 

```

- GitHub Secrets 配置

Secrets 是提供给 action 使用的安全变量机制，Secrets 中定义的变量都会进行加密，但后续可以在 action 中正常使用。

进入 Hugo 博客仓库，点击 `Settings`，左侧找到 `Secrets`，进入 `Secrets` 配置:

<img src="https://img1.kiosk007.top/static/images/bye_hexo/github_actions.png">

如图所示，点击右上角的 `New repository secret` 创建新变量：这里先创建私钥变量 `REMOTE_KEY`，在变量 `Name` 中输入 `REMOTE_KEY`。从服务器上拷贝私钥文件 `rsync` 的全部内容到 `Value` 中，点击 `Add secret` 保存：

使用同样的方式创建剩余 Secrets：

> `REMOTE_HOST`：远程主机地址。
> `REMOTE_PORT`：远程主机 SSH 端口，默认为 22 可不配置，若不是 22 必须配置。
> `REMOTE_USER`：数据同步使用的用户，如本文中使用的 work。
> `REMOTE_PATH`：远程主机上的目标路径，同步文件将会拷贝到该路径中，如 nginx 配置的静态网站路径。


## 配置 GitHub Actions
在仓库目录下创建 `.github/workflows` 目录，并且在目录中创建 `deploy.yml` 文件：
``` bash
- .github
  |- workflows
    |- deploy.yml
```
内容如下：
``` yml
name: deploy

on:
  # push事件
  push:
    # 忽略某些文件和目录，自行定义
    paths-ignore:
      - 'archetypes/**'
      - '.gitignore'
      - '.gitmodules'
      - 'README.md'
    branches: [ master ]

  # pull_request事件
  pull_request:
    # 忽略某些文件和目录，自行定义
    paths-ignore:
      - 'archetypes/**'
      - '.gitignore'
      - '.gitmodules'
      - 'README.md'
    branches: [ master ]
  
  # 支持手动运行
  workflow_dispatch:
    
jobs:
  # job名称为deploy
  deploy:
    # 使用GitHub提供的runner
    runs-on: ubuntu-20.04

    steps:
      # 检出代码，包括submodules，保证主题文件正常
      - name: Checkout source
        uses: actions/checkout@v2
        with:
          ref: master
          submodules: true  # Fetch Hugo themes (true OR recursive)
          fetch-depth: 0    # Fetch all history for .GitInfo and .Lastmod
      
      # 准备Hugo环境
      - name: Setup Hugo
        uses: peaceiris/actions-hugo@v2
        with:
          hugo-version: 'latest'
          # extended: true

      # Hugo构建静态站点，默认输出到public目录下    
      - name: Build
        run: hugo --gc --verbose --minify
        
      # 用docker容器上传algolia索引文件
      # - uses: actions/checkout@v2
      - name: Run container for atomic-algolia
        run: docker run --rm -e ALGOLIA_ADMIN_KEY=${{ secrets.ALGOLIA_ADMIN_KEY }} -e ALGOLIA_INDEX_FILE=/public/index.json -e ALGOLIA_APP_ID=${{ secrets.ALGOLIA_APP_ID }} -e ALGOLIA_INDEX_NAME=${{ secrets.ALGOLIA_INDEX_NAME }} -v $PWD/public:/public registry.cn-shenzhen.aliyuncs.com/lan-k8s/ubuntu:algolia atomic-algolia

      # 将public目录下的所有内容同步到远程服务器的nginx站点路径，注意path参数的写法，'public'和'public/'是不同的
      - name: Deploy
        uses: burnett01/rsync-deployments@5.1
        with:
          switches: -avzr --delete
          path: ./public/
          remote_host: ${{ secrets.REMOTE_HOST }}
          remote_port: ${{ secrets.REMOTE_PORT }}
          remote_path: ${{ secrets.REMOTE_PATH }}
          remote_user: ${{ secrets.REMOTE_USER }}
          remote_key: ${{ secrets.REMOTE_KEY }}

```
action 的 3 个 step 工作内容分别如下：

1. 先检出代码到 runner，包括 submodule 下的内容，这是为了保证作为 submodule 的主题目录能正常使用。
2. `actions-hugo@v2` 安装准备好 Hugo 的环境，with 表示 Hugo 版本为 latest。
3. `hugo --gc --verbose --minify` 构建 Hugo 静态站点，默认输出到 public 目录中。
4. `rsync-deployments@5.1` 将 public 目录中所有的内容全部拷贝到服务器指定路径，这里 `${{ secrets.XXX }}` 引用我们之前创建好的 `Secrets` 变量。


参考：
- [Hugo 部署——Github Actions](https://www.liyangjie.cn/posts/hobby/hugo-git-actions/)
- [自动构建algolia索引](https://liangyuanpeng.com/post/auto-build-algolia-index/)

