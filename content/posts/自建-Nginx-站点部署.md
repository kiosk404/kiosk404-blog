---
title: 自建 Nginx 部署
author: kiosk
tags:
  - nginx
  - devops
categories:
  - Nginx
date: 2020-04-18 20:51:00
---
这段时间搭建自己的 [Nginx](http://nginx.org/) 服务器，顺便总结回顾了一些运维相关的知识点，详细介绍了如何搭建一个企业级的高性能 Nginx 服务器。
<!--more-->

# 部署
## 部署环境
- 本机：Ubuntu 19.10
- Nginx：tengine-2.3.2
- 部署脚本：https://github.com/weijiaxiang007/nginx_install

``` bash
$ git clone https://github.com/weijiaxiang007/nginx_install.git
```


## Nginx 部署
> Nginx是一款自由的、开源的、高性能的HTTP服务器和反向代理服务器，也是今天的主角。


各组件版本
- luajit="LuaJIT-2.0.5"
- openssl="openssl-1.1.1f"
- pcre="pcre-8.44"
- tengine="tengine-2.3.2"
- jemalloc="jemalloc-5.2.1"

nginx 部署脚本 [install.sh](https://github.com/weijiaxiang007/nginx_install/blob/master/install.sh)

``` bash
$ sudo su
$ cd nginx_install
$ bash install.sh
```
默认安装用户为 work ，安装目录在 `/home/work/nginx`, 自定义虚拟主机可以放在 `/home/work/nginx/conf.d`。默认配置文件在`/home/work/nginx/conf/nginx.conf`。之后Nginx将作为一个出色的反向代理服务器，成为后面工作的主角。
``` bash
# nginx -V
JXWServer version: JXWServer/2.3.2
nginx version: nginx/1.17.3
built by gcc 9.2.1 20191008 (Ubuntu 9.2.1-9ubuntu2) 
built with OpenSSL 1.1.1f  31 Mar 2020
TLS SNI support enabled
configure arguments: --prefix=/home/work/nginx --error-log-path=/home/work/log/nginx/nginx.log --add-module=./modules/ngx_http_upstream_dyups_module --add-module=./modules/ngx_http_concat_module --add-module=./modules/ngx_http_upstream_session_sticky_module --add-module=./modules/ngx_http_upstream_check_module --add-module=./modules/ngx_http_upstream_dynamic_module --add-module=./modules/ngx_http_lua_module --add-module=./modules/ngx_backtrace_module --add-module=./modules/ngx_http_reqstat_module --add-module=./modules/ngx_http_user_agent_module --add-module=./modules/ngx_multi_upstream_module --add-module=./modules/headers-more-nginx-module --add-module=./modules/ngx_cache_purge --add-module=./modules/ngx_devel_kit --add-module=./modules/echo-nginx-module --add-module=./modules/ngx_slab_stat --add-module=./modules/ngx_http_sysguard_module --add-module=./modules/ngx_http_upstream_consistent_hash_module --add-module=./modules/ngx_http_proxy_connect_module --add-module=./modules/ngx_http_upstream_keepalive_module --without-http_upstream_keepalive_module --with-http_gunzip_module --with-http_gzip_static_module --with-http_ssl_module --with-http_v2_module --with-http_realip_module --with-http_addition_module --without-mail_pop3_module --without-mail_imap_module --without-mail_smtp_module --with-http_sub_module --with-pcre=/tmp/nginx_install/pcre-8.44 --with-pcre-jit --with-openssl=/tmp/nginx_install/openssl-1.1.1f --with-openssl-opt=enable-shared --with-jemalloc=/tmp/nginx_install/jemalloc-5.2.1 --with-luajit-inc=/usr/local/include/luajit-2.0 --with-luajit-lib=/usr/local/lib --with-http_auth_request_module --with-http_stub_status_module --with-google_perftools_module --with-ld-opt=-ltcmalloc --with-http_secure_link_module

```


## Bind 部署
> BIND是一种开源的DNS（Domain Name System）协议的实现，包含对域名的查询和响应所需的所有软件。它是互联网上最广泛使用的一种DNS服务器，对于类UNIX系统来说，已经成为事实上的标准。

我们搭建内网的dns服务器，不仅可以解析我们自定义的域名，也可以作为转发型\缓存型的DNS服务器解析其他公网域名。

部署版本
- bind9

安装bind9
``` bash
$ sudo apt-get update
$ sudo apt-get install bind9 bind9utils bind9-doc
```
bind9的配置文件默认在 `/etc/bind`。主配置文件被保存在下列文件中。
``` bash
/etc/bind/named.conf
/etc/bind/named.conf.options
/etc/bind/named.conf.local
```
我的部署脚本中有最简版的bind9配置文件模板，可以将 `./bind9/named.conf.options` 拷贝到 `/etc/bind/named.conf.options`。然后照着例子生成一个 `db.demo.com`的待解析的文件即可
对配置文件做下简单的介绍
``` bash
# cat named.conf.options
...
		forwarders {
                127.0.0.53;  // 对于公网域名，直接转发。并缓存记录
        };

        //========================================================================
        // If BIND logs error messages about the root key being expired,
        // you will need to update your keys.  See https://www.isc.org/bind-keys
        //========================================================================
        dnssec-validation auto;  // 递归查询服务器上开启DNSSEC验证

        max-cache-ttl 120;  
        max-ncache-ttl 120;
        version "[no version.]";
        minimal-responses yes;
        recursion yes;    // 允许递归查询，直接返回结果
        allow-query {any;};  // 允许所有主机查询
...

```

上述的例子是最简单的例子，没有引入复杂的视图，logging等概念，详情可以参考如下链接。如果这是本机访问的话，大可不必搭建bind服务器，可以直接写 `/etc/hosts`

More Info:
- [Bind9安装设置指南](https://wiki.ubuntu.org.cn/Bind9%E5%AE%89%E8%A3%85%E8%AE%BE%E7%BD%AE%E6%8C%87%E5%8D%97#Repositories_.E8.BD.AF.E4.BB.B6.E5.BA.93)
- [DNS(三)DNS SEC(域名系统安全扩展)](https://www.cnblogs.com/ibyte/p/5805548.html)
- [bind9支持edns-client-subnet](https://blog.csdn.net/zhiyuan_2007/article/details/48930449)
- [BIND简易教程（1）：安装及基本配置](https://www.cnblogs.com/anpengapple/p/5877661.html)

## 自签证书
为了能实现https访问当前的自定义的域名。需要自签证书，然后将自签证书导入到系统证书中。

自签证书工具
- openssl1.1.1k

自签证书已经写好了脚本只需要简单的运行即可生成自签证书
``` bash
$ cd ssl
$ bash create_ca.sh
$ bash create_client_cert.sh --ou SRE --cn kiosk.io --email admin@kiosk.io
```
ca的证书在 `./demoCA/cacert.pem`
kiosk.io 的证书在 `./kiosk.io/kiosk.io.crt` ， key在 `./kiosk.io/kiosk.io.key` 的。拷贝至Nginx所在目录即可。
将ca证书加入系统证书中。
```
$ sudo cp ./demoCA/cacert.pem /usr/local/share/ca-certificates/cacert.crt
$ sudo update-ca-certificates
```
其实update-ca-certificates是一个shell脚本,命令的本质其实是将PEM格式的根证书内容附加到**/etc/ssl/certs/ca-certificates.crt**, 而**/etc/ssl/certs/ca-certificates.crt** 中本身就包含了系统自带的各种可信根证书。

当然在企业内网还可以签署 客户端证书，没有装客户端证书的用户将不能访问该站点。在Nginx的配置文件中加上如下配置。
``` bash
  ssl_client_certificate ssl_certs/demoCA/cacert.pem;
  ssl_crl ssl_certs/demoCA/private/ca.crl;
  ssl_verify_client on;
```
> ssl_client_certificate就是客户端证书的CA证书了，代表此CA签发的证书都是可信的。
> ssl_verify_client on;代表强制启用客户端认证，非法客户端(无证书，证书不可信)都会返回400错。

特别注意ssl_crl这个配置，代表Nginx会读取一个CRL(Certificate Revoke List)文件，之前说过，可能会有收回用户权限的需求，因此我们必须有吊销证书的功能，产生一个CRL文件让Nginx知道哪些证书被吊销了即可。

需要收回签发证书时，只需要
``` bash
$ bash ./revoke_cert.sh aaa.demo.com
```
这个脚本会自动吊销他的签证文件crt，并且自动更新CRL文件。特别注意需要reload或restart nginx才能让nginx重新加载CRL。这样被吊销的证书将无法访问网站了。


**可能出现的问题**

- NET::ERR_CERT_COMMON_NAME_INVALID

自建CA和自签证书文档，但是发现自己生成之后，将ca证书导入客户端之后，Chrome访问网站总是会出现该错误。

如果要通过 修改 openssl.cnf 来签发证书，除将上述配置直接改到 openssl.cnf 相应位置外，必须将配置中的 basicConstraints = CA:FLASE 改为 basicConstraints = CA:TRUE，否则修改不生效，这是其他教程没有提到的。

如果是域名证书，也可以在此可以添加多域名，如：
``` bash
# vim http.ext
keyUsage = nonRepudiation, digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth, clientAuth
subjectAltName=@SubjectAlternativeName

[ SubjectAlternativeName ]
DNS.1=test.com
DNS.2=www.test.com
```
extendedKeyUsage 可以指定证书目的，即用途，一般有：

- serverAuth：保证远程计算机的身份
- clientAuth：向远程计算机证明你的身份
- codeSigning：确保软件来自软件发布者，保护软件在发行后不被更改
- emailProtection：保护电子邮件消息
- timeStamping：允许用当前时间签名数据

如果不指定，则默认为 所有应用程序策略

More Info:
- [Ubuntu添加可信任根证书](https://kiosk.io/admin/#/posts/ck95p6zs90000sfc6frdv25ia)
- [OpenSSL自签发自建CA签发SSL证书](https://www.cnblogs.com/will-space/p/11913744.html)
- [Nginx SSL快速双向认证配置(脚本)](https://blog.dteam.top/posts/2018-06/nginx-ssl%E5%BF%AB%E9%80%9F%E5%8F%8C%E5%90%91%E8%AE%A4%E8%AF%81%E9%85%8D%E7%BD%AE%E8%84%9A%E6%9C%AC.html)
- [如何将.pem转换为.crt和.key？](https://vimsky.com/article/3608.html)

# Nginx 配置文件
在`/home/work/nginx/conf.d/` 下新建`kiosk.io.conf`
配置文件内容如下：
``` nginx
upstream  hexo_backend {
        server 127.0.0.1:4000  weight=1 max_fails=2 fail_timeout=30s;      
}


server {
	listen 80;
	listen 443 ssl;
        server_name kiosk.io http2;

	if ($scheme != "https") {
    		return 301 https://$http_host$request_uri;
  	}
	
        access_log  /home/work/log/nginx/https_kiosk.io.log jxjson;
        error_log   /home/work/log/nginx/ssl-error.log notice;

        ssl_certificate         ssl/kiosk.io/kiosk.io.crt;
        ssl_certificate_key     ssl/kiosk.io/kiosk.io.key;
        ssl_prefer_server_ciphers   on;

        ssl_session_cache  shared:SSL:80m;
        ssl_session_timeout  5m;
        ssl_protocols   TLSv1.1 TLSv1.2 TLSv1.3;
        ssl_ciphers EECDH+AESGCM:EDH+AESGCM:AES256+EECDH:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-RSA-AES256-SHA384:ECDHE-RSA-AES128-SHA256:ECDHE-RSA-AES256-SHA:ECDHE-RSA-AES128-SHA:ECDHE-RSA-DES-CBC3-SHA:DHE-RSA-AES256-GCM-SHA384:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-SHA256:DHE-RSA-AES128-SHA256:DHE-RSA-AES256-SHA:DHE-RSA-AES128-SHA:DHE-RSA-DES-CBC3-SHA:AES256-GCM-SHA384:AES128-GCM-SHA256:AES256-SHA256:AES128-SHA256:AES256-SHA:AES128-SHA:DES-CBC3-SHA:HIGH:!aNULL:!eNULL:!EXPORT:!DES:!MD5:!PSK:!RC4;
        ssl_stapling on; 
        ssl_stapling_verify on; 
        ssl_trusted_certificate ssl/kiosk.io/kiosk.io.crt;
        ssl_dhparam ssl/dhparam.pem;
                
        ssl_early_data     on;
        add_header Strict-Transport-Security "max-age=31536000; includeSubdomains";
        add_header X-Content-Type-Options nosniff;

        proxy_set_header   Early-Data $ssl_early_data;
        proxy_set_header   Host  $host;
        proxy_set_header   X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header   X-Real-IP         $remote_addr;
        proxy_set_header   X-Forwarded-Proto $scheme;
	
	location / {
            proxy_pass http://hexo_backend/;
        }
	
	error_page   500 502 503 504  /50x.html;
        proxy_intercept_errors on;
        location = /50x.html {
               root   html;
        }

}

```


日志格式参考: https://adamtheautomator.com/nginix-logs/