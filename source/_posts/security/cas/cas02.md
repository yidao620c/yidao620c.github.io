---
title: CAS教程 - 服务安装
date: 2019-01-03 10:22:12 +0800
toc: true
categories: [ 网络安全 ]
tags: [ CAS ]
---

这篇开始写一个系列，讲解如何使用配置和使用CAS实现单点登录。

我的环境：

* CAS-server：5.3.X
* Maven：3.6.0
* JDK：1.8
* cas-server域名：cas.server.com
* tomcat服务器：SpringBoot内置/tomcat8
* 操作系统：华为云主机上CentOS 7

Cas支持Gradle和Maven两种编译方式，使用SpringBoot构建，我这里使用Maven的方式。
<!-- more -->

## Maven镜像配置

华为和Sonatype 联合发布中国官方 Maven 仓库，特意试了试，感觉速度挺快的，比阿里的快。这里贴一下配置：

```xml

<servers>
    <server>
        <id>huaweicloud</id>
        <username>anonymous</username>
        <password>devcloud</password>
    </server>
</servers>

<mirrors>
<mirror>
    <id>nexus-aliyun</id>
    <mirrorOf>central</mirrorOf>
    <name>Nexus aliyun</name>
    <url>http://maven.aliyun.com/nexus/content/groups/public</url>
</mirror>
<mirror>
    <id>huaweicloud</id>
    <mirrorOf>*</mirrorOf>
    <name>Nexus osc thirdparty</name>
    <url>https://mirrors.huaweicloud.com/repository/maven/</url>
</mirror>
</mirrors>
```

## 下载

Maven地址：https://github.com/apereo/cas-overlay-template

Gradle地址：https://github.com/apereo/cas-gradle-overlay-template

我下载的是5.3分支：

```bash
git clone https://github.com/apereo/cas-overlay-template.git -b 5.3
```

## 构建

将下载Overlay解压，默认配置就可以直接构筑能用的 war 包：

```bash
cd cas-overlay-template/
./build.sh package
```

## 证书生成

由于CAS服务器需要HTTPS访问，所以需要生产证书并导入JRE中。

创建一个cas目录，cd进入cas目录之后执行如下命令

注意要点

* 名字和姓氏输入项：访问的域名地址
* alias：别名可以随便写，一般要有意义，后续操作别名要一致，我这里保持和域名统一了
* 密码：changeit
* 别名：cas.server.com

**使用java自带keytool创建本地密钥库**

```bash
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

```bash
[root@ecs-hw01 cas]# keytool -export -alias cas.server.com -keystore casServer.keystore -file casServer.crt -storepass changeit
Certificate stored in file <casServer.crt>

Warning:
The JKS keystore uses a proprietary format. It is recommended to migrate to PKCS12 which is an industry standard format using "keytool -importkeystore -srckeystore casServer.keystore -destkeystore casServer.keystore -deststoretype pkcs12".
```

**查看证书**

```bash
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

```bash
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

```bash
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

```bash
keytool -delete -alias cas.server.com -keystore /usr/local/jdk/jre/lib/security/cacerts
```

## 修改配置

第一步，先将上面创建的证书文件 `casServer.crt`、`casServer.keystore` 拷贝到项目的/etc/cas/目录中。

第二步，然后修改`cas.properties`文件

```properties
cas.server.name:https://cas.server.com:8443
cas.server.prefix:https://cas.server.com:8443/cas
cas.adminPagesSecurity.ip=127\.0\.0\.1
logging.config:file:/etc/cas/config/log4j2.xml
# cas.serviceRegistry.config.location: classpath:/services
server.ssl.keyStore=file:/etc/cas/casServer.keystore
server.ssl.keyStorePassword=changeit
server.ssl.keyPassword=changeit
server.ssl.key-alias=cas.server.com
```

第三步，修改log4j2.xml 日志文件

在`cas-overlay-template-master`项目中，创建一个logs目录，用于存放日志文件，
默认cas在项目根目录下生成日志，日志多了不方便管理，修改log4j2.xml文件，将日志目录做调整。

比如将`fileName="${sys:cas.log.dir}/cas.log"` 修改为 `fileName="${sys:cas.log.dir}/logs/cas.log"`

第四步，将项目的/etc/cas复制到操作系统的/etc/命令中

```bash
[root@ecs-hw01 cas-overlay-template]# cp -r etc/cas/ /etc/
```

## 运行测试

```bash
[root@ecs-hw01 cas-overlay-template]# ./build.sh run
```

如果是windows本地开发就在terminal中执行`build run`

本地修改host文件

```
xx.xx.xx.xx cas.server.com
```

访问地址：<https://cas.server.com:8443/cas>

默认账号：`casuser`默认密码：`Mellon`

![](https://xnstatic-1253397658.file.myqcloud.com/cas20190221-01.png)

登录成功后页面：

![](https://xnstatic-1253397658.file.myqcloud.com/cas20190221-02.png)

退出登录：

![](https://xnstatic-1253397658.file.myqcloud.com/cas20190221-03.png)

## tomcat部署

上面演示的是SpringBoot内置容器运行方式，现在演示如何部署到已有的tomcat容器中。

tomcat的下载和安装我就不讲了，直接下载tomcat包解压即可。

然后把上面mvn package命令生成的cas.war复制到webapps/目录中。

**配置SSL**

修改tomcat的配置文件server.xml，找到下面的代码：

```xml

<Connector port="8443" protocol="org.apache.coyote.http11.Http11NioProtocol"
           maxThreads="150" SSLEnabled="true">
    <SSLHostConfig>
        <Certificate certificateKeystoreFile="/etc/cas/casServer.keystore"
                     certificateKeystoreType="JKS" certificateKeystorePassword="changeit"/>
    </SSLHostConfig>
</Connector>
```

完了后去bin目录执行start.sh即可。访问：<https://cas.server.com:8443/cas>

效果一样，完成验证！
