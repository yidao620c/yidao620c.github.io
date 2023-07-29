---
title: 发布Maven构件到中央仓库
date: 2018-01-27 12:09:12 +0800
toc: true
categories: [ 开发工具 ]
tags:
  - maven
---

之前写过一篇如何使用Nexus私服，发布自己写的maven构件，供大家使用。但是只能在公司内部用，
而你想全世界的人都能用到你写的东西，就需要发布到Maven中央仓库了。

本篇文章详细讲解如何发布Maven构件到中央仓库。
<!-- more -->

## 注册Sonatype的账户

maven中央仓库是有一个叫做Sonatype的公司在维护的，在发布构件之前需要 [注册一个账号](https://issues.sonatype.org/secure/Signup!default.jspa)，
记住自己的用户名和密码，以后要用。

同时，还要记住一个地址，将来在查询自己所发布构件状态和进行一些操作的时候要使用 <https://oss.sonatype.org/>

## 提交发布申请

提交申请，在这里是创建一个issue的形式，创建地址：<https://issues.sonatype.org/secure/CreateIssue.jspa?issuetype=21&pid=10134>

在填写issue信息的时候，有一些需要注意的地方：

1. "group id" 就是别人在使用你的构件的时候在pom.xml里面进行定位的坐标的一部分，最好是自己的域名倒序，
   如果自己没有域名就填写github域名，比如`io.github.yidao620c`
2. "project url"是这个项目站点，填写你的github项目地址即可。
3. "SCM url"这个是构件的源代码URL，一般就是可以通过`git clone`命令来下载源码的地址，
   比如我的是 `https://github.com/yidao620c/springfox-swagger-ui.git`

其他的就没有什么了，提交之后就等工作人员离开确认吧。有时候你填写的是自己域名的时候，工作人员会问你是不是真的是自己的域名，
你在下面回答他就可以了。

![](https://xnstatic-1253397658.file.myqcloud.com/maven19.png)

过段时间就去看看你的issue，如果工作人员回复你说"Configuration has been prepared..."，如下：

![](https://xnstatic-1253397658.file.myqcloud.com/maven20.png)

那么你就可以开始下一步了。

## GPG签名

在上传构件之前，需要准备GPG以便对发布的文件进行签名。

windows用户到 <http://www.gpg4win.org/download.html> 去下载Gpg4win-Vanilla版来使用，linux的直接安装gpg软件包就行。

下载之后执行：

```bash
gpg --gen-key
```

需要输入姓名、邮箱等字段，其它字段可使用默认值，此外，还需要输入一个 Passphase，相当于一个密钥库的密码，
一定不要忘了，也不要告诉别人，最好记下来，因为后面会用到。

## 配置settings.xml

找你所使用的maven的配置文件<mvn_home>/conf/settings.xml，在配置文件中找到<servers>节点，
这个节点默认是注释了的，我们就在这个猪似的外边增加一个<servers>的配置如下：

```xml
<servers>
    <server>
        <id>oss</id>
        <username>用户名</username>
        <password>密码</password>
    </server>
</servers>
```

这里的id是将来要在pom.xml里面使用的，所以务必记好，用户名和密码就是在Sonatype上面注册的用户名和密码。

## 配置pom.xml

接下来就是重头戏了，pom.xml是一个maven项目的重点配置，一个项目的所有配置都可以由这个文件来描述，
文件中的所有配置都有默认值，也就是说所有的配置都是可选配置，但是为了把构件发布到中央仓库，
我们必须配置一些关键信息，否则再发布时是不会通过了。

这些必须明确的信息包括：name、description、url、licenses、developers、scm等基本信息，
此外，使用了 Maven 的 profile 功能，只有在 release 的时候，创建源码包、创建文档包、使用 GPG 进行数字签名。
此外，snapshotRepository 与 repository 中的 id 一定要与 settings.xml 中 server 的 id 保持一致。

下面是我的一个示例：

```xml

<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.xncoding</groupId>
    <artifactId>springfox-swagger-ui</artifactId>
    <version>2.7.0</version>
    <packaging>jar</packaging>

    <name>springfox-swagger-ui</name>
    <description>swagger-ui for xncoding</description>
    <url>https://www.xncoding.com/</url>

    <licenses>
        <license>
            <name>The MIT License (MIT)</name>
            <url>https://opensource.org/licenses/MIT</url>
        </license>
    </licenses>

    <developers>
        <developer>
            <name>yidao620c</name>
            <email>yidao620@gmail.com</email>
        </developer>
    </developers>

    <scm>
        <connection>scm:git:git@github.com:yidao620c/springfox-swagger-ui.git</connection>
        <developerConnection>scm:git:git@github.com:yidao620c/springfox-swagger-ui.git</developerConnection>
        <url>https://github.com/yidao620c/springfox-swagger-ui</url>
    </scm>

    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
        <maven.compiler.encoding>UTF-8</maven.compiler.encoding>
        <java.version>1.8</java.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>io.springfox</groupId>
            <artifactId>springfox-spring-web</artifactId>
            <version>2.7.0</version>
            <scope>compile</scope>
        </dependency>
    </dependencies>

    ...其他省略...

    <profiles>
        <profile>
            <id>release</id>
            <build>
                <plugins>
                    <!-- Source -->
                    <plugin>
                        <groupId>org.apache.maven.plugins</groupId>
                        <artifactId>maven-source-plugin</artifactId>
                        <version>3.0.1</version>
                        <executions>
                            <execution>
                                <phase>package</phase>
                                <goals>
                                    <goal>jar-no-fork</goal>
                                </goals>
                            </execution>
                        </executions>
                    </plugin>
                    <!-- Javadoc -->
                    <plugin>
                        <groupId>org.apache.maven.plugins</groupId>
                        <artifactId>maven-javadoc-plugin</artifactId>
                        <version>2.10.4</version>
                        <executions>
                            <execution>
                                <phase>package</phase>
                                <goals>
                                    <goal>jar</goal>
                                </goals>
                            </execution>
                        </executions>
                    </plugin>
                    <!-- GPG -->
                    <plugin>
                        <groupId>org.apache.maven.plugins</groupId>
                        <artifactId>maven-gpg-plugin</artifactId>
                        <version>1.6</version>
                        <executions>
                            <execution>
                                <phase>verify</phase>
                                <goals>
                                    <goal>sign</goal>
                                </goals>
                            </execution>
                        </executions>
                    </plugin>
                </plugins>
            </build>
            <distributionManagement>
                <snapshotRepository>
                    <id>oss</id>
                    <url>https://oss.sonatype.org/content/repositories/snapshots/</url>
                </snapshotRepository>
                <repository>
                    <id>oss</id>
                    <url>https://oss.sonatype.org/service/local/staging/deploy/maven2/</url>
                </repository>
            </distributionManagement>
        </profile>
    </profiles>
</project>
```

## 上传构件

待构件编写完成，就可以进行上传、发布了。在命令行进入项目pom.xml所在路径，执行：

```bash
mvn clean deploy -P release
```

在稍后些时候会要你输入gpg密钥库的密码，输入即可完成上传。
当然有时候不会弹出输入密码的输入框，只是提示需要输入密码，根据gpg插件的官网解释，需要加上密码作为参数执行命令，即：

```bash
mvn clean deploy -P release -Dgpg.passphrase=密码
```

## 在OSS中发布构件

构建上传之后需要在OSS系统中对操作进行确认，将构件发布，进入 <https://oss.sonatype.org/> 使用你的用户名和密码登陆之后，
在左边菜单找到"Staging Repositories"，点击，在右边上面一点有一个输入搜索框输入你的groupid进行快速定位，
可以发现这时你的构件状态是"open"，勾选你的构件，查看校验的结果信息，
如果没有错误就可以点击刚才勾选的checkbox上面右边一点的"close"按钮，在弹出框中"confirm"，这里又需要校验一次，稍后结果会通过邮箱通知。

我再close过程中出现过验证签名失败的问题：

```
 No public key: Key with id: (xxxx) was not able to be located on http://keyserver.ubuntu.com:11371
```

只需要在本地执行：

```bash
gpg --keyserver hkp://keyserver.ubuntu.com --send-keys xxxx
```

其中的xxxx是你的GPG key，在生成GPG密钥的时候控制台会打印。

发送key完成后再次试一下close成功了。刷新页面，这是状态已经是"closed"的了，再次勾选，然后点击"close"旁边的"release"，
在弹出框中进行"confirm"，稍后结果会通过邮件进行通知。

注意，你执行release成功之后，OSS里面就没有这个构件了，我一开始还在想这家伙跑哪去了，原来是跑到Maven中央库里面去啦。

## 通知sonatype的工作人员关闭issue

回到issue系统，找到你的那个申请发布构件的issue，在下面回复工作人员，说明构件已经发布，待工作人员确认后，会关闭这个issue。

![](https://xnstatic-1253397658.file.myqcloud.com/maven21.png)

## 使用构件

一切完成后并不可以马上就使用你所发布的构件，得等系统将你的构件同步到中央仓库之后才可以使用，
这个时间至少要2个小时，然后就可以在中央仓库的搜索页面 <http://search.maven.org/> 搜到你的构件啦，

赶快截图，向他人炫耀一下吧。

![](https://xnstatic-1253397658.file.myqcloud.com/maven22.png)

## 下次再发布

你看了上面这长篇大论，感觉好像流程很复杂。但是好消息是你只需要第一次的时候这么做。后面再发布就轻松多啦。

第一次成功之后，以后就可以使用你的groupid发布任何的构件了，只需要你的groupid没有变就行，（当然不能发布重复构件哈），不用这么麻烦。

以后的发布流程：

1. 构件准备好之后，在命令行上传构建；
1. 登录 <https://oss.sonatype.org/> , close 并 release 构件；
1. 等待同步好（大约2小时多）之后，就可以使用了

