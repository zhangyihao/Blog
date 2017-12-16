---
title: 使用OpenSSL制作SSL证书
date: 2017-12-13 21:57:17
tags:
	- SSL证书
categories:
  - 安全认证
comments: false

---

制作SSL证书前，需要先了解两个概念：证书（certificate）和证书请求（certificate sign rquest）
> 1. 证书是自签名或CA签名过的凭据，用来进行身份认证。
> 2. 证书请求是对签名的请求，需要使用私钥进行签名。


## 1. 生成CA证书 ##
第一步，生成 CA 私钥：

```
[root@py ssl_test]# openssl genrsa -out ca.key 1024
```
- ca.key 为私钥文件名;
- 1024为密钥长度，单位为bits，不写即为默认值为512。
*openssl genrsa* 命令详细参数可参考维基百科：https://wiki.openssl.org/index.php/Manual:Genrsa(1)

<!--more-->

第二步，生成证书请求即 Certificate Signing Request (CSR)：

```
[root@py ssl_test]# openssl req -new -key ca.key -out ca.csr
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [XX]:BJ
State or Province Name (full name) []:beijing
Locality Name (eg, city) [Default City]:beijing
Organization Name (eg, company) [Default Company Ltd]:MYCA
Organizational Unit Name (eg, section) []:ca
Common Name (eg, your name or your server's hostname) []:ca
Email Address []:

Please enter the following 'extra' attributes
to be sent with your certificate request
A challenge password []:123456
An optional company name []:
```
填写上边信息时需要注意：
- *-key* 参数指定了私钥文件，即第一步生成的文件。
- *Organization Name (eg, company) [Default Company Ltd]:* 需要填写单位组织名称，后面生成客户端和服务器端证书的时候也需要填写，不要写成一样的，可以随意写程：MyCA, MyServer, MyClient等，区别开即可。
- *Common Name (eg, your name or your server's hostname) []* ：在生成服务器请求证书时，此处需要填写为服务器ip地址或域名；此时生成的为CA请求证书，可以随意填写。
- 以上填写信息，可通过 *-subj* 参数指定 或这 *-batch*参数读取配置文件。 

*openssl req* 命令详细参数可参考维基百科：https://wiki.openssl.org/index.php/Manual:Req(1)


第三步，生成证书 Certificate（CRT）：

```
[root@py ssl_test]# openssl x509 -req -days 365 -in ca.csr -signkey ca.key -out ca.crt
```
- *-days* 参数指定了证书有效期，单位为天，默认为30天。 
- *-in* 指定输入文件，即证书请求文件。
- *-signkey* 自签名证书需要的私钥文件。
命令详细参数可参考维基百科：https://wiki.openssl.org/index.php/Command_Line_Utilities

## 2. 生成服务器端证书 ##
第一步，生成服务器端公钥、私钥：

```
生成服务器私钥：
[root@py ssl_test]# openssl genrsa -out server.key 1024
生成服务器公钥：
[root@py ssl_test]# openssl rsa -in server.key -pubout -out server.pem
```
第二步，生成服务器端证书请求文件

```
[root@py ssl_test]# openssl req -new -key server.key -out server.csr
```
注意，此时 *Common Name* 要填写服务器的ip地址或域名地址。

第三步，向CA机构申请证书
在这一步，CA机构使用CA的证书和私钥，对请求的服务器请求证书进行签名，最后生成一个带有CA签名的证书。

```
[root@py ssl_test]# openssl x509 -req -days 365 -CA ca.crt -CAkey ca.key -CAcreateserial -in server.csr -out server.crt
```

## 3. 生成客户端证书 ##
生成方式及命令，参考上边生成服务端证书部分。

### 将客户端证书文件转换为p12格式证书文件 ###
```
[root@py ssl_test]# openssl pkcs12 -export -clcerts -in client.crt -inkey client.key -out clien.p12
```