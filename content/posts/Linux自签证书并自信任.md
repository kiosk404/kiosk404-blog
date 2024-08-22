---
title: Linux自签证书并自信任
author: kiosk
tags:
  - ssl
categories:
  - Nginx
date: 2024-08-04 10:16:00
---





由于本地自建了Nginx，需要HTTPS访问，所以这里记录一下 Linux 下的 证书自签名和自信任。



<!--more-->



操作环境：Ubuntu 24.04

假设我要签署的服务器域名 kiosk.io

# 自签证书

1. 安装 openssl

```bash
sudo apt-get install openssl -y
```

2. 生成 RootCA 的私钥

```bash
openssl genrsa -out ca.key 2048
```

3. 创建一个名为`ca.cnf`的配置文件，其中包含以下内容：

```bash
[ req ]
default_bits = 2048
prompt = no
default_md = sha256
distinguished_name = dn

[ dn ]
C = CN
ST = Beijing
L = Beijing
O = AAAOrganization
OU = AAA
CN = AAA

[ v3_ca ]
subjectAltName = @AAA

[ alt_names ]
DNS.1 = ssl-aaa.com
```

然后使用该配置文件创建CA证书：

```bash
openssl req -x509 -new -nodes -key ca.key -sha256 -days 3650 -out ca.crt -config ca.cnf
```

4. 生成 kiosk.io 服务器私钥

```bash
openssl genrsa -out kiosk.io.key 2048
```

5. 创建 kiosk.io 服务器域名的证书签名请求(CSR):

> `openssl.cnf` 是一个 OpenSSL 配置文件，它包含了用于证书签名请求（CSR）、证书和证书颁发机构（CA）的设置。一个完整的 `openssl.cnf` 文件可能包含多个部分，用于定义不同的配置选项。以下是一个基本的 `openssl.cnf` 文件示例，它包含了用于生成自签名 CA 证书和服务器证书的配置：

```bash
[ req ]
default_bits = 2048
prompt = no
distinguished_name = req_distinguished_name
req_extensions = v3_req

[ req_distinguished_name ]
C = CB
ST = Beijing
L = Beijing
O = Kiosk Organization
OU = KioskUnit
CN = www.kiosk.io

[ v3_req ]
basicConstraints = CA:TRUE
keyUsage = nonRepudiation, digitalSignature, keyEncipherment, dataEncipherment
subjectAltName = @alt_names

[ alt_names ]
DNS.1 = www.kiosk.io
DNS.2 = kiosk.io
```

创建

```bash
openssl req -new -key kiosk.io.key -out kiosk.io.csr -reqexts v3_req -config openssl.cnf
```

7. 使用 CA 证书签署服务器 kiosk.io 的 CSR

```
openssl x509 -req -in kiosk.io.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out kiosk.io.crt -days 365 -extensions v3_req -extfile openssl.cnf
```

8. 验证证书

```
openssl verify -CAfile ca.crt kiosk.io.crt
```

<br/>

# 证书自信任

正常情况下，本地的 Chrome 浏览器是不信任的， 如下图

<img src="https://img1.kiosk007.top/static/images/blog//20240822231649-截图 2024-08-22 23-16-04.png" />

### Linux系统：

在Linux系统中，添加CA证书通常涉及到更新系统的证书库。以下是在Ubuntu和其他基于Debian的系统上的步骤：

1. **将CA证书复制到证书库目录**：

   ```
   sudo cp /path/to/your/ca.crt /usr/local/share/ca-certificates/
   ```

   这里的`/path/to/your/ca.crt`是你的CA证书文件的路径。

2. **更新证书库**：

   ```
   sudo update-ca-certificates
   ```

   这个命令会更新系统的证书库，并自动将新添加的证书包含进去。

3. 更新 Chrome 浏览器的根证书

```
chrome -> 设置 -> 隐私设置和安全性 -> 管理证书 -> 导入
```

