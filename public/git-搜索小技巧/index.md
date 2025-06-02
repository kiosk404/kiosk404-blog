# Github 搜索

本着绝不重复造论子的理念（其实就是想抄袭），需要在github上这个平台搜索自己感兴趣的项目。但是在平常使用中，其实经常没有搜索到自己真正想要的git项目。
<!--more-->

---

# github的核心功能

- **features**
<img src="https://img1.kiosk007.top/static/images/git/github_feature.png" />

github 的核心功能可以查看 [features
](https://github.com/features) ，github的核心功能主要分为7大类 `持续继承/持续交付(CI/CD)`、`安全部署（Secure development）`、`代码检查（Code review）`、`Git应用（Apps）`、`代码托管 (Hosting)`、`项目管理 （Project managment）`、`团队管理（Team managment）`

- **developer**
<img src="https://img1.kiosk007.top/static/images/git/github_develop.png" />

github 的API可以查看 [api](https://developer.github.com/) , 比如可以获取github上的自己的repo列表、OAuth2 token 认证、自动推送修改repo中文件等。

refer: https://segmentfault.com/a/1190000015144126

# 搜索git项目

搜索项目可以在 Market Place 中找到一些火热的项目、工具等

- **Market Place**

<img src="https://img1.kiosk007.top/static/images/git/github_marketplace.png" />

地址：https://github.com/marketplace/

里面有诸如监控、持续继承、devops的项目，大部分是免费的，可以免费安装使用。
如：

- [Sentry](https://github.com/marketplace/sentry) ：提供客户端APP实时的崩溃报告的平台。
- [Jira Software + GitHub](https://github.com/marketplace/jira-software-github) ：将代码项目和项目管理平台Jira连接起来的平台。
- [gitpod.io](https://github.com/marketplace/gitpod-io) : Github在线IDE gitpod.io

**搜索技巧**

点击github左上角的搜索框，输入`Enter键`，弹出github搜索页，点击`advance search` 高级搜索。

<img src="https://img1.kiosk007.top/static/images/git/advance_search.png" />

高级搜索中可以按编程语言搜索，按repo时间搜索等等。

<img src="https://img1.kiosk007.top/static/images/git/search_date.png" />

默认直接在左上角的输入，只是搜索整个github所有repo的标题和描述。而我们其实要找的内容一般在readme中。

- 搜索readme

``` git
arp 欺骗 in:readme
```

<img src="https://img1.kiosk007.top/static/images/git/search_readme.png" />


- 搜索star大于100

``` git
arp 欺骗 in:readme stars:>100
```

- 搜索go语言

``` git
arp 欺骗 in:readme stars:>10 language:Go
```
