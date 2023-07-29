---
title: IDEA不能下载maven依赖包的源码
date: '2018-06-03 15:12:09 +0800'
toc: true
categories: [ 开发工具 ]
tags: [ idea, maven ]
---

有时候在IDEA里面直接点击查看源码，报：cannot download sources

使用Maven命令。经过测试，好用。下载了所有POM里的依赖包的source，这点不是想要的，原来只想下载想看的依赖的source。
参考：[IDEA-165800 Can’t download dependency's source code](https://youtrack.jetbrains.com/issue/IDEA-165800)
<!-- more -->

使用如下命令行下载：

```bash
mvn dependency:resolve -Dclassifier=sources
```

如果只想下载指定的包，使用（多个使用逗号分隔）：

```bash
mvn dependency:sources -DincludeArtifactIds=commons-io,:mybatis
```

参考：[Get source JARs from Maven repository](http://stackoverflow.com/questions/2059431/get-source-jars-from-maven-repository)

