---
title: Maven传递依赖无法引入解决办法
date: '2018-11-02 12:09:17 +0800'
toc: true
categories: [ 开发工具 ]
tags: [ maven ]
---

今天一个传递依赖问题搞了我半天，终于搞明白原因了。一个jar包A依赖了httpclient，然后另一个jar包B引入A，
在IDEA里面只能看到依赖A，不管咋样都看不到依赖httpclient。

我在IDEA的项目B里面，打包后在控制台发现一个告警：

```
the POM for A is invalid, transitive dependencies (if any) will not be available, 
enable debug logging for more details
```

原来是jar包A的pom依赖有问题。
<!-- more -->

## 问题排查

在应用B的根目录打印依赖树：

```bash
mvn dependency:tree>tree.txt
```

应用依赖树中出现如下警告。警告显示：应用引入的依赖包无效，依赖包中传递依赖项不可用，可以通过开启debug获取更多信息。

```
[WARNING] the POM for A is invalid, transitive dependencies (if any) will not be available, enable debug logging for more details... 
```

然后我开启debug功能，重新打印依赖树：

```bash
mvn -X dependency:tree>tree.txt
```

开启maven debug功能后，警告后紧跟了一条错误信息，如下。

```
[WARNING] The POM for com.huawei.dc.security:security-sdk-https:jar:1.0-SNAPSHOT is invalid, transitive dependencies (if any) will not be available: 2 problems were encountered while building the effective model for com.huawei.dc.security:security-sdk-https:1.0-SNAPSHOT
[ERROR] 'dependencies.dependency.version' for org.springframework.boot:spring-boot-configuration-processor:jar is missing. @ 
[ERROR] 'dependencies.dependency.version' for org.springframework.boot:spring-boot-autoconfigure:jar is missing. @ 
...
```

原来是A包中引入的另外的spring-boot-autoconfigure没加版本号，这个我以为能继承父pom的版本号，发现这里不生效。具体原因我再分析。

## 解决方案

在A中把版本号补上重新发布，则一切正常。
