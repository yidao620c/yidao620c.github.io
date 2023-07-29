---
title: 基于keepalived实现mysql双主
date: 2016-09-21 12:17:29 +0800
toc: true
categories: [中间件]
tags: [ mysql ]
---

MySQL服务端开启SSL保证连接通道的安全性，同时可支持开启证书双向认证，进一步提升安全。

<!-- more -->

## 服务端配置

这里通过openssl命令来生成自签名证书。

第1步，生成CA证书：

```bash
openssl genrsa 2048 > ca-key.pem
openssl req -new -x509 -nodes -days 3650 -key ca-key.pem >ca-cert.pem
```

第2步，通过CA证书签发服务证书：

```bash
openssl req -newkey rsa:2048 -days 3650 -nodes -keyout server-key.pem > server-req.pem
openssl x509 -req -in server-req.pem -days 3650 -CA ca-cert.pem -CAkey ca-key.pem \
-set_serial 01 > server-cert.pem
openssl rsa -in server-key.pem -out server-key.pem
```

第3步，通过CA证书签发客户端证书：

```bash
openssl req -newkey rsa:2048 -days 3650 -nodes -keyout client-key.pem > client-req.pem
openssl x509 -req -in client-req.pem -days 3650 -CA ca-cert.pem -CAkey ca-key.pem \
-set_serial 01 > client-cert.pem
openssl rsa -in client-key.pem -out client-key.pem
```

第4步，修改MySQL服务配置`/etc/my.cnf`，添加或更新如下内容：

```ini
[mysqld]
ssl-ca=ca-cert.pem
ssl-cert=server-cert.pem
ssl-key=server-key.pem

[client]
ssl-cert=client-cert.pem
ssl-key=client-key.pem
```

然后重启MySQL服务即完成了服务端的配置。

## 客户端配置

这里说明Java的JDBC客户端怎样通过SSL方式连接MySQL，并开启证书双向认证。

![](https://xnstatic-1253397658.file.myqcloud.com/20201118_mysql01.png)

``` java
// SSL Mode
properties.put("sslMode",SSLModeType.VERIFY_CA.name());

// Setting the Username, Password
        properties.put("user",USER);
        properties.put("password",PASSWORD);

// Setting the truststore
        properties.put("trustCertificateKeyStoreUrl","file:path_to_truststore_file");
        properties.put("trustCertificateKeyStorePassword","mypassword");

// Setting the keystore
        properties.put("clientCertificateKeyStoreUrl","file:path_to_keystore_file");
        properties.put("clientCertificateKeyStorePassword","mypassword");
```

## 证书校验模式

总共5中校验模式，安全等级逐级提升。

1. No（sslMode=DISABLED）

不开启SSL连接通道，直接明文传输。

2. Preferred（sslMode=PREFERRED）

自动先尝试用SSL连接，如果MySQL服务端不支持SSL就退回到明文传输。不校验证书和主机名。

3. Require（sslMode=REQUIRED）

总是使用SSL连接，如果MySQL服务端不支持SSL，连接中断。不校验证书和主机名。

4. Require and Verify CA（sslMode=VERIFY_CA）

总是使用SSL连接，如果MySQL服务端不支持SSL，连接中断。同时会校验服务端证书，但是不校验主机名。

5. Require and Verify Identity（sslMode=VERIFY_IDENTITY）

总是使用SSL连接，如果MySQL服务端不支持SSL，连接中断。同时会校验服务端证书和主机名。
