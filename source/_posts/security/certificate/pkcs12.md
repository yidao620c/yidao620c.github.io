---
title: 【安全贴士】PKCS12弱加密算法分析
date: 2020-05-01 12:33:08 +0800
toc: true
categories: [ 网络安全 ]
tags:
  - 数字证书
  - 安全贴士
---

在CentOS7和JDK8中，默认使用的openssl或keytool命令生成的PKCS12证书库， 
使用的加密算法为弱加密算法`pbeWithSHA1And40BitRC2-CBC`，需要升级为新的安全算法。

可通过如下命令查看PKCS12格式的证书库文件详细信息。
<!-- more -->

```bash
[root@mail CA]# openssl pkcs12 -info -in root78.p12 -nodes
MAC Iteration 100000
MAC verified OK
PKCS7 Encrypted data: pbeWithSHA1And40BitRC2-CBC, Iteration 50000
Certificate bag
Bag Attributes
    friendlyName: caroot
    2.16.840.1.113894.746875.1.1: <Unsupported tag 6>
subject=/C=CN/ST=SX/L=XA/O=xncoding/OU=GTS/CN=ca.xncoding.com/emailAddress=ca@xncoding.com
issuer=/C=CN/ST=SX/L=XA/O=xncoding/OU=GTS/CN=ca.xncoding.com/emailAddress=ca@xncoding.com
-----BEGIN CERTIFICATE-----
MIIFHTCCA4WgAwIBAgIJAPr+QX/LXhniMA0GCSqGSIb3DQEBCwUAMIGCMQswCQYD
```

## 升级方案

openssl命令可以通过参数来指定加密算法

```bash
# 导出包含CA链的PKCS12格式证书库
openssl pkcs12 -export -inkey private/server.key -passin pass:${server_key_password} \
-in certs/server.crt -chain -CAfile root.crt -out certs/server.p12 -password pass:${server_p12_password} \
-keypbe aes-256-cbc -certpbe aes-256-cbc

# 导出信任证书库
openssl pkcs12 -export -nokeys -in root.crt -certfile root.crt -out root.p12 -certpbe AES-256-CBC
```

JDK8目前并不能解析这种加密方式的证书库，因此需要添加开源库`bouncycastle`，Java中加载证书库的时候指定provider为`BC`才能解析。
方法是在代码中添加`Security.addProvider(new BouncyCastleProvider())`。

或者更好的做法是在jre的`java.security`文件中添加

```
security.provider.10=org.bouncycastle.jce.provider.BouncyCastleProvider
```

另外如果在Tomcat配置HTTPS双向认证，则同样需要指定证书库和信任证书库的provider为`BC`。

## JDK版本问题

按照上面做法做之后，仍然无法解决问题。使用这种方式导出来的`server.p12`证书库是没有问题的，但是如果使用这种方式导出信任证书`root.p12`，则无法使用。 经过反复测试后发现，由于`root.p12`
缺少别名导致，而openssl无法对每个证书单独设置别名。事实上，openssl对于信任证书库的支持很有限。 因此无法通过openssl命令来生成、导入、导出、删除信任证书库等操作，只能通过keytool命令。

对于keytool命令，需要使用高版本的JDK来导出信任证书库，具体参考：<https://bugs.openjdk.java.net/browse/JDK-8228481>。
根据bug说明，对于JDK8、11、15、16、17都解决了，但是实测发现8和11都不行，只有OpenJDK 15支持。主要是这个补丁版还没发布吧。 等后续再看看是否已修复。

方法就是下载OpenJDK 15之后，修改jre中的java.security

```
keystore.pkcs12.certProtectionAlgorithm=PBEWithHmacSHA256AndAES_256
keystore.pkcs12.keyProtectionAlgorithm=PBEWithHmacSHA256AndAES_256
keystore.pkcs12.macAlgorithm=HmacPBESHA256
keystore.pkcs12.certPbeIterationCount=10000
keystore.pkcs12.keyPbeIterationCount=10000
keystore.pkcs12.macIterationCount=100000
```

然后使用OpenJDK 15的keytool工具生成信任证书库`root.p12`。

```bash
keytool -import -noprompt -trustcacerts -alias caroot -file root.crt -keystore root.p12 -storetype PKCS12
```

也就是说证书`server.p12`由`openssl`命令生成，而信任证书库`root.p12`由`keytool`工具生成。 生成之后对于证书的解析方法仍然是需要使用`BC`。

