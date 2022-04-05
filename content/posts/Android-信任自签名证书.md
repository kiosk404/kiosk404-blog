---
title: Android 信任自签名证书
author: kiosk
tags:
  - Certificate
categories:
  - Linux
date: 2020-07-22 23:23:00
---
在之前的文章 [VMTrafficShark 自制弱网模拟器](https://kiosk007.top/2020/05/30/VMTrafficShark-%E8%87%AA%E5%88%B6%E5%BC%B1%E7%BD%91%E6%A8%A1%E6%8B%9F%E5%99%A8/) 中介绍了如何搭建一个弱网模拟器，并且进行弱网模拟，中间人，劫持重放测试等等。对了这里一般的测试设备是Android手机，那么这篇文章介绍一下Android 手机应该如何支持自签名证书。

<!--more-->

-------------------------

# Android root
Android 本身可以理解为一个Linux操作系统的终端，只是没有root权限，普通用户 `adb` 连上之后只有普通用户的权限。而root不是那么容易的，这里推荐 `Magisk Root` 这款软件，目前我的 Mi3， RedMi Note 8 Pro 都是已经root了的（曾经是米粉...）

甩几个链接参考

- [Magic Mask To Enhance Your Default Android | The First Systemless Root Tool](https://www.magiskroots.com/)
- [How To Setup Magisk Manager On Rooted Android Device To Hide “Root”](https://devsjournal.com/how-to-setup-magisk-manager-on-rooted-android-device-to-hide-root.html#:~:text=Installing%20Magisk%20Root%20via%20TWRP%3A%201%20Make%20sure,Zip%20and%20confirm%20flashing.%20...%20More%20items...%20)
- [How to Root Xiaomi Redmi Note 5](https://c.mi.com/thread-1808315-1-0.html)


# 生成自签名证书
## X509 证书简介

首先介绍一下证书体系吧

公钥基础设施（PKI）是一种用于验证网络世界中的用户和设备的技术。基本思想是让一个或多个受信方对数据进行数字签名，以证明特定的加密密钥属于特定的用户或设备。然后该密钥可以用作证明该用户的身份。

而 x509 正是当下最常见的证书标准，[RFC5280](https://tools.ietf.org/html/rfc5280) 阐述了IETF对X509证书的定义。x509证书的格式在google chrome 浏览器地址栏左边的小锁即可查看。

以cloudflare的官网域名 https://1.1.1.1 举例（没错，还有ip证书）

**第一组详细信息包括有关主题的信息，包括公司名称和地址以及证书旨在保护的网站的通用名称（Fully Qualified Domain Name）**
<img src='https://img1.kiosk007.top/static/images/network/AndroidRoot/cert_fqdn.png' style='height:450px' />

**向下滑动，是发行者的信息（Issuer）, 这里可以看到签发者是 DigiCert。在颁发者下方，我们可以看到证书的序列号，X.509版本（3），签名算法以及指定证书有效期的日期。**
<img src='https://img1.kiosk007.top/static/images/network/AndroidRoot/cert_sign.png' style='height:450px' />

**X.509 v3证书还包括一组扩展，这些扩展在证书使用方面提供了更多的灵活性。例如，主题备用名称扩展名允许证书绑定到多个身份。（因此，有时将多域证书称为 [SAN](https://www.ssl.com/faqs/what-is-a-san-certificate/) 证书）。这里提供了很多其他域名，这里还可以提供ip地址,可以看到 cloudflare 的1.1.1.1**

<img src='https://img1.kiosk007.top/static/images/network/AndroidRoot/cert_san.png' style='height:450px' />

## 证书格式
- .DER .CER，文件是二进制格式，只保存证书，不保存私钥。
- .PEM，一般是文本格式，可保存证书，可保存私钥。
- .CRT，可以是二进制格式，可以是文本格式，与 .DER 格式相同，不保存私钥。
- .PFX .P12，二进制格式，同时包含证书和私钥，一般有密码保护。
- .JKS，二进制格式，同时包含证书和私钥，一般有密码保护。

## 利用 Mitm 的 Root CA 制作自签名证书

[mitmproxy](https://docs.mitmproxy.org/stable/) 是一款中间人工具，通过代理来实现https流量嗅探。

安装好 mitmproxy 后再家目录下就放着 mitm 的root ca。
``` bash
➜  ls ~/.mitmproxy
mitmproxy-ca-cert.cer  mitmproxy-ca-cert.p12  mitmproxy-ca-cert.pem  mitmproxy-ca.p12  mitmproxy-ca.pem  mitmproxy-dhparam.pem

```

这里我们取 `mitmproxy-ca.pem` 这个文件制作自签名证书。

### 分离CA证书和私钥

``` bash
➜  mkdir mitm && cd mitm
➜  cp ~/.mitmproxy/mitmproxy-ca.pem .

# 分离私钥
➜  openssl pkey -in mitmproxy-ca.pem -out mitmproxy-ca-private.key

# 提取所有证书，包括CA Chain
➜  openssl crl2pkcs7 -nocrl -certfile mitmproxy-ca.pem | openssl pkcs7 -print_certs -out mitmproxy-ca-certificate.cert
```

### 创建自签名证书
``` bash
# 生成证书私钥（your-domain.com 换成你要签名的域名）
➜  openssl genrsa -out your-domain.com.key 2048

# 生成证书请求文件(并按要求填写 CN、SPN、LN等，不想填就全部回车)
➜  openssl req -new -key your-domain.com.key -out your-domain.com.csr

```
在 Chrome 58 之后，改为使用 SAN(Subject Alternative Name) 检查域名的一致性。

而 SAN 属于 x509 扩展里面的内容，所以我们需要通过 -extfile 参数来指定存放扩展内容的文件。创建一个 your-domain.com.ext 文件用来保存 SAN 信息，通过指定多个 DNS 从而可以实现多域名证书。
参考：[FAQ/subjectAltName (SAN)](http://wiki.cacert.org/FAQ/subjectAltName)

``` bash
➜  vim your-domain.com.ext
[req]
req_extensions = v3_req

[v3_req]
authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
subjectAltName = @alt_names

[alt_names]
DNS.1 = your-domain.com.
DNS.2 = *.your-domain.com.
IP.1 = 127.0.0.1
IP.2 = 192.168.0.107
```
使用 mitm 的CA 公私钥对 签署自己域名的证书。
``` bash
➜  openssl x509 -req -sha256 -in your-domain.com.csr \
-CA mitmproxy-ca-certificate.cert \
-CAkey mitmproxy-ca-private.key \
-CAcreateserial \
-out your-domain.com.crt \
-days 365 \
-extfile kiosk007.top.ext \
-extensions v3_req

```

至此，自签名证书已经生成完毕，自签名的证书为 `your-domain.crt` , 自签名的私钥为 `your-domain.key` 。只需要将 `mitmproxy-ca-certificate.cert` 导入到Android手机中，手机便会信任由这个root 证书签名的证书，如刚才的 `your-domian.crt`

## 导入根证书到Android手机

### 修改文件名
``` bash
# 得到hash值
➜  openssl x509 -subject_hash_old -in mitmproxy-ca-certificate.cert |head -n 1
➜  cp mitmproxy-ca-certificate.cert <Certificate_Hash>.0

```

### 安装到Android机
``` bash
➜  adb push c8750f0d.0 /sdcard/
➜  adb shell

# 手机中操作
begonia:/ $ su root
:/ # mount -o rw,remount /
:/ # mv /sdcard/c8750f0d.0 /system/etc/security/cacerts/
:/ # chmod 644 /system/etc/security/cacerts/c8750f0d.0
:/ # chown root:root /system/etc/security/cacerts/c8750f0d.0
:/ # mount -o ro,remount /
:/ # reboot
➜  
➜  
```

> 如果遇到'/dev/block/dm-0' is read-only报错，依次执行以下命令解决：
> 1. adb root
> 2. adb disable-verity
> 3. adb reboot
> 4. adb remount
> 5. adb shell 
> 6. mount -o rw,remount /system

> 如若还不行(Andoird11)，可以进fastboot模式，执行 `fastboot oem unlock`

参考：
- [What Is an X.509 Certificate?](https://www.ssl.com/faqs/what-is-an-x-509-certificate/)
- [PKI - Public Key Infrastructure](https://www.ssh.com/pki/)
- [SSL 证书格式普及，PEM、CER、JKS、PKCS12](https://blog.freessl.cn/ssl-cert-format-introduce/)


# Android ssh 终端 (附加)

这一步主要是可以通过ssh操作终端（已经和信任自签名证书无关了哈，只是附加的。哈哈），因为`adb shell`实在是太难操作了。

这里先安装 Termux （可以在 google play 或者 豌豆荚下载）

安装 Termux 之后，安装sshd
手机上操作：
``` 
pkg install openssh -y
# start ssh
sshd
```

配置远程登录
```
# PC copy ssh 公钥 (没有的化，执行 ssh-keygen 生成)
➜  copy id_rsa.pub

# Termux 粘贴公钥
cd .ssh
vim authorized_keys
粘贴保存

查看当前用户，后面登录用
whoami

# PC 上配置端口转发
（如果是远程的话配置一下 adb connect xxxxxx:xxx）
➜  adb forward --list
➜  adb forward --remove-all
➜  adb forward tcp:8022 tcp:8022
➜  ssh u0_a221@127.0.0.1 -p 8022

```
> Tips: 界面左滑，长按 `KEYBOARD` 可以出现辅助按钮， ESC、CTR、ALT 等。再次单按 `KEYBOARD` 可出现键盘。

<img src="https://img1.kiosk007.top/static/images/network/AndroidRoot/android_root.png" />