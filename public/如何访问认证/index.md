# 如何进行访问认证？


最近有个Web项目需要对接公司的权限认证系统来保证应用的数据安全隔离。于是对访问认证来了兴趣，这里进行一次统一的梳理。



<!--more-->

# 基本的认证方式

在梳理之前先要搞清楚什么是认证和授权。

- **认证（Authentication，英文缩写authn）**：用来验证某个用户是否具有访问系统的权限，如果认证通过，该用户就可以访问系统，从而创建、修改、删除、查询平台支持的资源。
- **授权（Authorization，英文缩写authz）**：用来验证某个用户是否具有访问某个资源的权限，如果授权通过，该用户就能对资源做增删查改等操作。

常见的认证方式有四种，分别是 Basic、Digest、OAuth 和 Bearer。

<br/>

## Basic

Basic 认证（基础认证），是最简单的认证方式。它简单地将用户名:密码进行 base64 编码后，放到 HTTP Authorization Header 中。HTTP 请求到达后端服务后，后端服务会解析出 Authorization Header 中的 base64 字符串，解码获取用户名和密码，并将用户名和密码跟数据库中记录的值进行比较，如果匹配则认证通过。例如：

```bash
$ basic=`echo -n 'admin:Admin@2022'|base64`
$ curl -XPOST -H"Authorization: Basic ${basic}" http://127.0.0.1:8080/login
```

认证过程足够简单，但是极不安全。线上环境如果需要使用必须搭配 HTTPS 来使用。



<br/>

## Digest

为弥补 BASIC 认证存在的弱点，从 HTTP/1.1 起就有了 DIGEST 认证。

<img src="https://img1.kiosk007.top/static/images/blog/auth.png" alt="auth" style="zoom:50%;;clear:both;display:block;margin:auto" />

1. 客户端请求服务端的资源。
2. 在客户端能够证明它知道密码从而确认其身份之前，服务端认证失败，返回401 Unauthorized，并返回WWW-Authenticate头，里面包含认证需要的信息。
3. 客户端根据WWW-Authenticate头中的信息，选择加密算法，并使用密码随机数 nonce，计算出密码摘要 response，并再次请求服务端。
4. 服务器将客户端提供的密码摘要与服务器内部计算出的摘要进行对比。如果匹配，就说明客户端知道密码，认证通过，并返回一些与授权会话相关的附加信息，放在 Authorization-Info 中。

WWW-Authenticate 中内容如下：

| 字段                  | 说明                                                      |
| --------------------- | --------------------------------------------------------- |
| username              | 用户名                                                    |
| realm                 | 服务器返回的realm,一般是域名                              |
| method                | HTTP 请求方法                                             |
| nonce                 | 随机字符串                                                |
| nc（nonceCount）      | 请求次数，用于标记、计数、防止重放攻击                    |
| cnonce（clientNonce） | 客户端发送给服务器的随机字符串，用于客户端对服务器的认证  |
| qop                   | 保护质量参数，一般是 auth 或 auth-int，这回影响摘要的算法 |
| uri                   | 请求的uri                                                 |
| response              | 客户端根据算法算出的密码摘要值                            |





<br/>

## OAuth

OAuth（开放授权）是一个开放授权的标准，允许用户让第三方应用访问该用户在某一 Web 服务上的资源而无需将自己的用户名和密码提供给第三方应用，OAuth 目前是 2.0 版本。

- **密码式**



- **隐藏式**



- **凭借式**



- **授权码模式**



<br/>

## Bearer

Bearer 认证也称为 令牌认证，是一种 HTTP 身份验证方法。Bearer 认证的核心是 bearer token。bearer token 是一个加密字符串，通常由服务器根据密钥生成。客户端在请求服务端时，必须在请求头包含 `Authorization：Bearer <token>` ，服务端受到请求后，解析出 `<token>`  ，并校验其合法性。

当前最流行的 token 编码方式是 JSON Web Token（[JWT](https://jwt.io/)) 



<br/>
