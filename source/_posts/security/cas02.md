---
title: "CAS服务安装"
date: 2019-01-03 10:22:12 +0800
comments: true
toc: true
categories: 网络安全
tags: [CAS]
---

这篇开始写一个系列，讲解如何使用配置和使用CAS实现单点登录。

我的环境：

* CAS-server：5.2.X
* Maven：3.6.0
* JDK：1.8
* cas-server域名：cas.server.com
* tomcat服务器：我用的SpringBoot插件
* 操作系统：华为云主机上CentOS 7

Cas支持Gradle和Maven两种方式部署，使用SpringBoot构建，我这里使用Maven的方式。<!--more-->

## 下载

Maven地址：https://github.com/apereo/cas-overlay-template

Gradle地址：https://github.com/apereo/cas-gradle-overlay-template

我下载的是5.2分支：

```
git clone https://github.com/apereo/cas-overlay-template.git -b 5.2
```

## 构建

将下载Overlay解压，默认配置就可以直接构筑能用的 war 包：

```
cd cas-overlay-template/
./build.sh package
```

## 证书生成

由于CAS服务器需要HTTPS访问，所以需要生产证书并导入JRE中。

创建一个cas目录，cd进入cas目录之后执行如下命令

注意要点

* 名字和姓氏输入项：访问的域名地址
* alias：别名可以随便写，一般要有意义，后续操作别名要一致，我这里保持和域名统一了

**使用java自带keytool创建本地密钥库**

```
[root@ecs-hw01 cas]# keytool -genkey -alias cas.server.com -keyalg RSA -keystore casServer.keystore
Enter keystore password:  
Re-enter new password: 
What is your first and last name?
  [Unknown]:  cas.server.com
What is the name of your organizational unit?
  [Unknown]:  wusong
What is the name of your organization?
  [Unknown]:  yanfazu
What is the name of your City or Locality?
  [Unknown]:  beijing
What is the name of your State or Province?
  [Unknown]:  dongcheng
What is the two-letter country code for this unit?
  [Unknown]:  zh
Is CN=cas.server.com, OU=wusong, O=yanfazu, L=beijing, ST=dongcheng, C=zh correct?
  [no]:  y

Enter key password for <cas.server.com>
	(RETURN if same as keystore password):  

Warning:
The JKS keystore uses a proprietary format. It is recommended to migrate to PKCS12 which is an industry standard format using "keytool -importkeystore -srckeystore casServer.keystore -destkeystore casServer.keystore -deststoretype pkcs12".
```

**把密钥库导出成证书文件**

```
[root@ecs-hw01 cas]# keytool -export -alias cas.server.com -keystore casServer.keystore -file casServer.crt -storepass changeit
Certificate stored in file <casServer.crt>

Warning:
The JKS keystore uses a proprietary format. It is recommended to migrate to PKCS12 which is an industry standard format using "keytool -importkeystore -srckeystore casServer.keystore -destkeystore casServer.keystore -deststoretype pkcs12".
```

**查看证书**

```
[root@ecs-hw01 cas]# keytool -printcert -file casServer.crt
Owner: CN=cas.server.com, OU=wusong, O=yanfazu, L=beijing, ST=dongcheng, C=zh
Issuer: CN=cas.server.com, OU=wusong, O=yanfazu, L=beijing, ST=dongcheng, C=zh
Serial number: 6f53fb64
Valid from: Sun Feb 17 20:52:57 CST 2019 until: Sat May 18 20:52:57 CST 2019
Certificate fingerprints:
	 MD5:  9F:B9:29:69:B5:15:AF:B2:4B:5D:97:06:EA:C7:3C:F3
	 SHA1: 3D:64:ED:8C:FA:AF:F2:B6:E5:D5:A6:5D:DB:7F:44:EF:84:EC:8C:F7
	 SHA256: EF:A5:8A:B1:28:89:4C:D6:4E:B6:AF:64:F2:4A:A0:2B:B5:7B:4D:3E:9A:60:57:F3:6A:37:89:0C:FF:19:D9:BF
Signature algorithm name: SHA256withRSA
Subject Public Key Algorithm: 2048-bit RSA key
Version: 3

Extensions: 

#1: ObjectId: 2.5.29.14 Criticality=false
SubjectKeyIdentifier [
KeyIdentifier [
0000: F9 B6 E1 F4 5F 3B 8D FE   88 1E 59 6A 2D 68 8D B1  ...._;....Yj-h..
0010: 66 4D E1 AF                                        fM..
]
]
```

**将创建过的证书导入到java证书库**

```
[root@ecs-hw01 cas]# keytool -import -keystore "/usr/local/jdk/jre/lib/security/cacerts" -file "/root/cas/casServer.crt" -alias cas.server.com
Enter keystore password:  
Owner: CN=cas.server.com, OU=wusong, O=yanfazu, L=beijing, ST=dongcheng, C=zh
Issuer: CN=cas.server.com, OU=wusong, O=yanfazu, L=beijing, ST=dongcheng, C=zh
Serial number: 6f53fb64
Valid from: Sun Feb 17 20:52:57 CST 2019 until: Sat May 18 20:52:57 CST 2019
Certificate fingerprints:
	 MD5:  9F:B9:29:69:B5:15:AF:B2:4B:5D:97:06:EA:C7:3C:F3
	 SHA1: 3D:64:ED:8C:FA:AF:F2:B6:E5:D5:A6:5D:DB:7F:44:EF:84:EC:8C:F7
	 SHA256: EF:A5:8A:B1:28:89:4C:D6:4E:B6:AF:64:F2:4A:A0:2B:B5:7B:4D:3E:9A:60:57:F3:6A:37:89:0C:FF:19:D9:BF
Signature algorithm name: SHA256withRSA
Subject Public Key Algorithm: 2048-bit RSA key
Version: 3

Extensions: 

#1: ObjectId: 2.5.29.14 Criticality=false
SubjectKeyIdentifier [
KeyIdentifier [
0000: F9 B6 E1 F4 5F 3B 8D FE   88 1E 59 6A 2D 68 8D B1  ...._;....Yj-h..
0010: 66 4D E1 AF                                        fM..
]
]

Trust this certificate? [no]:  y
Certificate was added to keystore
```

**查看JDK证书内容**

```
[root@ecs-hw01 cas]# keytool -list -v -keystore /usr/local/jdk/jre/lib/security/cacerts -alias cas.server.com
Enter keystore password:  
Alias name: cas.server.com
Creation date: Feb 17, 2019
Entry type: trustedCertEntry

Owner: CN=cas.server.com, OU=wusong, O=yanfazu, L=beijing, ST=dongcheng, C=zh
Issuer: CN=cas.server.com, OU=wusong, O=yanfazu, L=beijing, ST=dongcheng, C=zh
Serial number: 6f53fb64
Valid from: Sun Feb 17 20:52:57 CST 2019 until: Sat May 18 20:52:57 CST 2019
Certificate fingerprints:
	 MD5:  9F:B9:29:69:B5:15:AF:B2:4B:5D:97:06:EA:C7:3C:F3
	 SHA1: 3D:64:ED:8C:FA:AF:F2:B6:E5:D5:A6:5D:DB:7F:44:EF:84:EC:8C:F7
	 SHA256: EF:A5:8A:B1:28:89:4C:D6:4E:B6:AF:64:F2:4A:A0:2B:B5:7B:4D:3E:9A:60:57:F3:6A:37:89:0C:FF:19:D9:BF
Signature algorithm name: SHA256withRSA
Subject Public Key Algorithm: 2048-bit RSA key
Version: 3

Extensions: 

#1: ObjectId: 2.5.29.14 Criticality=false
SubjectKeyIdentifier [
KeyIdentifier [
0000: F9 B6 E1 F4 5F 3B 8D FE   88 1E 59 6A 2D 68 8D B1  ...._;....Yj-h..
0010: 66 4D E1 AF                                        fM..
]
]
```

**根据 alias 别名删除 JDK 证书**

做完实验后如果你想删除JDK证书，可使用如下命令：
```
keytool -delete -alias cas.server.com -keystore /usr/local/jdk/jre/lib/security/cacerts
```

## 修改配置

第一步，先将上面创建的证书文件 casServer.crt、casServer.keystore 拷贝到项目的/etc/cas/目录中。

第二步，然后修改cas.properties文件

```
cas.server.name: https://cas.server.com:8443
cas.server.prefix: https://cas.server.com:8443/cas

cas.adminPagesSecurity.ip=127\.0\.0\.1

logging.config: file:/etc/cas/config/log4j2.xml
# cas.serviceRegistry.config.location: classpath:/services

server.ssl.keyStore=file:/etc/cas/casServer.keystore
server.ssl.keyStorePassword=changeit
server.ssl.keyPassword=changeit
server.ssl.key-alias=cas.server.com
```

第三步，修改log4j2.xml 日志文件

在cas-overlay-template-master项目中，创建一个logs目录，用于存放日志文件，
默认cas在项目根目录下生成日志，日志多了不方便管理，修改log4j2.xml文件，将日志目录做调整。

比如将`fileName="${sys:cas.log.dir}/cas.log"` 修改为 `fileName="${sys:cas.log.dir}/logs/cas.log"`

第四步，将项目的/etc/cas复制到操作系统的/etc/命令中

```
[root@ecs-hw01 cas-overlay-template]# cp -r etc/cas/ /etc/
```

## 运行测试

```
[root@ecs-hw01 cas-overlay-template]# ./build.sh run
```

本地修改host文件

```
xx.xx.xx.xx cas.server.com
```

访问地址：<https://cas.server.com:8443/cas>

默认账号：`casuser`默认密码：`Mellon` 

![](https://xnstatic-1253397658.file.myqcloud.com/cas20190217-01.jpg)

登录成功后页面：

![](https://xnstatic-1253397658.file.myqcloud.com/cas20190217-02.jpg)

退出登录：

![](https://xnstatic-1253397658.file.myqcloud.com/cas20190217-03.jpg)


