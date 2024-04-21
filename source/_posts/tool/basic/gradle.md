---
title: Gradle构建入门
date: 2017-12-10 10:22:29 +0800
toc: true
categories: [ 开发工具 ]
tags: [ 开发工具 ]
---

在开发Spring Boot应用的时候，除了Maven以外，Gradle也是一个不错的构建选择。
IDEA 默认使用Gradle工具做Android的构建程序，在开发APP的时候就必须要使用到Gradle了。
这里简单记录一下使用的方法，以及跟IDEA的集成方式。
<!-- more -->

## 下载和安装

以Windows为例

安装groovy语言，这一步可以省略，不过为了方便最好还是装上了。

按照[Groovy官方文档](http://groovy-lang.org/install.html)，先下载对应的二进制分发包，
解压到某个文件夹目录比如`E:\libs\`下面，然后配置环境变量和PATH：

```bash
GROOVY_HOME=E:\libs\groovy-2.4.13
PATH=%PATH%;%GROOVY_HOME%\bin
```

按照[Gradle官网向导](https://gradle.org/install/)手动安装。

先去下载最新二进制发布包：<https://gradle.org/releases/>

解压缩到某个盘比如`E:\libs\`下面，然后配置环境变量和PATH：

```bash
GRADLE_HOME=E:\libs\gradle-4.4
PATH=%PATH%;%GRADLE_HOME%\bin
```

如果你之前安装过maven私服，那就配置一下全局镜像为私服地址，如果没有就配置成阿里云的镜像。

将下面这段Copy到名为`init.gradle`文件中，并保存到 `USER_HOME/.gradle/`文件夹下即可：

```
allprojects {
    repositories {
        // def REPOSITORY_URL = 'http://maven.aliyun.com/nexus/content/groups/public/'
        def REPOSITORY_URL = 'http://123.207.66.156:8081/repository/maven-public/'
        all { ArtifactRepository repo ->
            if (repo instanceof MavenArtifactRepository && repo.url != null) {
                def url = repo.url.toString()
                if (url.startsWith('https://repo1.maven.org/maven2') || url.startsWith('https://jcenter.bintray.com/')) {
                    project.logger.lifecycle "Repository ${repo.url} replaced by $REPOSITORY_URL."
                    remove repo
                }
            }
        }
        maven {
            url REPOSITORY_URL
        }
    }
}
```

`init.gradle`文件其实是Gradle的初始化脚本(Initialization Scripts)，也是运行时的全局配置。
更详细的介绍请参阅 <http://gradle.org/docs/current/userguide/init_scripts.html>

## IDEA中使用

IDEA默认就支持Gradle，如果创建新的工程，只需要在向导中选择Gradle工程即可，后面的跟maven一样。
如果是导入已经存在的Gradle工程，就选择 `New` -> `Project from existing sources`，然后选中`build.gradle`即可。

