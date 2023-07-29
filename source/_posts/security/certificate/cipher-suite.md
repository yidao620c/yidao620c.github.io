---
title: 【安全贴士】加密算法套件CipherSuite介绍
date: 2020-05-19 02:10:22 +0800
toc: true
categories: [ 网络安全 ]
tags:
  - 数字证书
  - 安全贴士
---

`CipherSuite`这个名词目前没看到有好的中文翻译，个人觉得翻译成`加密算法套件`比较合适。 Cipher泛指是密码学的加密算法，例如 `aes`, `rsa`, `ecdh` 等。
tls是由各类基础算法作为原语组合而成，一个CipherSuite是4个算法的组合：

* 1个key exchange(密钥交换)算法
* 1个authentication(身份认证)算法
* 1个encryption(加密)算法
* 1个message authentication code (消息认证码，简称MAC)算法

比如一个例子`TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256`。 表示的含义是TLS_ECDHE（密钥交换算法）_RSA（身份认证算法）_WITH_AES_128_GCM（AES128GCM对称加密算法）
_SHA256（完整性包含的MAC算法）。
<!--more-->

## CipherSuite的注册管理

TLS Cipher Suite 在 iana 集中注册，每一个CipherSuite分配有一个2字节的数字用来标识。可以在iana的网页查看：
<https://www.iana.org/assignments/tls-parameters/tls-parameters.xhtml#tls-parameters-4>

例如:

```
0x00,0x2F   TLS_RSA_WITH_AES_128_CBC_SHA       Y    [RFC5246]
0x00,0x30   TLS_DH_DSS_WITH_AES_128_CBC_SHA    Y    [RFC5246]
0x00,0x31   TLS_DH_RSA_WITH_AES_128_CBC_SHA    Y    [RFC5246]
0x00,0x32   TLS_DHE_DSS_WITH_AES_128_CBC_SHA   Y    [RFC5246]
0x00,0x33   TLS_DHE_RSA_WITH_AES_128_CBC_SHA   Y    [RFC5246]
0x00,0x34   TLS_DH_anon_WITH_AES_128_CBC_SHA   Y    [RFC5246]
0x00,0x35   TLS_RSA_WITH_AES_256_CBC_SHA       Y    [RFC5246]
0x00,0x36   TLS_DH_DSS_WITH_AES_256_CBC_SHA    Y    [RFC5246]
0x00,0x37   TLS_DH_RSA_WITH_AES_256_CBC_SHA    Y    [RFC5246]
0x00,0x38   TLS_DHE_DSS_WITH_AES_256_CBC_SHA   Y    [RFC5246]
0x00,0x39   TLS_DHE_RSA_WITH_AES_256_CBC_SHA   Y    [RFC5246]
0x00,0x3A   TLS_DH_anon_WITH_AES_256_CBC_SHA   Y    [RFC5246]
0x00,0x3B   TLS_RSA_WITH_NULL_SHA256           Y    [RFC5246]
0x00,0x3C   TLS_RSA_WITH_AES_128_CBC_SHA256    Y    [RFC5246]
0x00,0x3D   TLS_RSA_WITH_AES_256_CBC_SHA256    Y    [RFC5246]
0x00,0x3E   TLS_DH_DSS_WITH_AES_128_CBC_SHA256 Y    [RFC5246]
0x00,0x3F   TLS_DH_RSA_WITH_AES_128_CBC_SHA256 Y    [RFC5246]
```

例如其中的：

```
0x00,0x37   TLS_DH_RSA_WITH_AES_256_CBC_SHA    Y    [RFC5246]
```

这一条，`0x00,0x37` 就是 `TLS_DH_RSA_WITH_AES_256_CBC_SHA` 这个 Cipher Suite的数字， rfc5246 定义了这个CipherSuite的具体实现。

## 使用CipherSuite

在tls中，选择CipherSuite的方法是通过cipher list

格式和用法见：<https://www.openssl.org/docs/apps/ciphers.html>

nginx里面的配置项是 `cipher_list`

`cipher list`的格式由一个或者多个由冒号分隔的`cipher string`(逗号和空格也可以接受但不常用)。

一个`cipher string`可以是下列形式之一:

1. 可以由单个`cipher suite`构成，例如 `RC4-SHA`。
2. 它可以表示含有某个特定算法的cipher列表，或者一种特定类型的`cipher suite`。 例如，SHA1表示所有使用摘要算法SHA1的`cipher suite`, SSLV3表示所有SSL V3算法。
3. `cipher suite`的列表,可以使用加号+合并到一个单一的`cipher string`里面。 这被作为一个逻辑且操作。例如，SHA1+DES表示所有包含了 SHA，并且包含了DES的算法。
4. 每一个`cipher string`可以在前面加上字符 !,-,或者+

> 如果加了!，那么这种cipher永久从列表里面删除，就算后边显式添加进来也不行。
>
> 如果加了-，那么cipher中的一些或者全部可以在后面的选项里面加回来。
>
> 如果加了+，那么cipher被移动到列表的最后，这个选项不增加任何cipher，只是把匹配的cipher移动到最后。
>
> 如果没有上述字符，那么字符串被解析成一个cipher list，追加到当前配置列表的后面。
>
> 如果cipher list 中的某些cipher已经存在了，就忽略该cipher。

5. 另外，`@STRENGTH`可以用在任何点，用来把当前`cipher list`按照加密算法key长度排序。

## 检测服务端支持的密钥套件脚本

可通过openssl命令，写一个通用脚本来检测服务端支持的所有SSL/TLS密钥套件。 使用方式`./test_ciphers 192.168.1.20:443`

```bash
#!/usr/bin/env bash

# OpenSSL requires the port number.
SERVER=$1
DELAY=1
ciphers=$(openssl ciphers 'ALL:eNULL' | sed -e 's/:/ /g')

echo Obtaining cipher list from $(openssl version).

for cipher in ${ciphers[@]}
do
echo -n Testing $cipher...
result=$(echo -n | openssl s_client -cipher "$cipher" -connect $SERVER 2>&1)
if [[ "$result" =~ ":error:" ]] ; then
  error=$(echo -n $result | cut -d':' -f6)
  echo NO \($error\)
else
  if [[ "$result" =~ "Cipher is ${cipher}" || "$result" =~ "Cipher    :" ]] ; then
    echo YES
  else
    echo UNKNOWN RESPONSE
    echo $result
  fi
fi
sleep $DELAY
done
```
