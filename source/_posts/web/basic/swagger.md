---
title: 使用Swagger生成RESTful API文档
date: 2017-06-09 12:32:08 +0800
toc: true
categories: [ web ]
tags: [ web ]
---

REST API都是要对外提供服务的，那么文档是必须的。Swagger是一个简单但功能强大的API表达工具。
它具有地球上最大的API工具生态系统，数以千计的开发人员，使用几乎所有的现代编程语言，
都在支持和使用Swagger。使用Swagger生成API，我们可以得到交互式文档，自动生成代码的SDK以及API的发现特性等。

2.X版本已经发布，Swagger变得更加强大。值得感激的是，Swagger的源码100%开源在[github](https://github.com/swagger-api)。

使用Swagger不纯粹是为了生成一个漂亮的API文档，也不纯粹是为了自动生成多种语言的代码框架，
重要的是，通过遵循它的标准，可以使REST API分组清晰、定义标准。
<!-- more -->

通过Swagger生成API 文档有两种方式：

1. 通过代码注解来生成。好处：随时保持接口和文档的同步。坏处：代码入侵
2. 使用`Swagger Editor` 编写API文档的Yaml/Json定义。

虽然第一种方式最方便，不用编写swagger配置文件，但是对代码污染太严重了。所以在项目里面我选择第二种方式，
另外我也不实用Swagger UI来展示API文档，页面太花哨了。
这里我选择swagger2markup将其转换为AsciiDoc或MarkDown格式。

## Swagger Editor使用

Swagger Editor是个用Angular开发的WEB小程序，它可以让你用YAML来定义你的接口规范，并实时验证和现实成接口文档。

此外，它还可以通过接口文档帮你生成不同框架的服务端和客户端，方便你mock和契约测试。
最后导出JSON格式的API规范，通过Swagger UI对外发布。

先clone项目：

```bash
git clone https://github.com/swagger-api/swagger-editor.git
```

如果在本地，解压后直接打开里面的index.html即可。

如果在服务器上面，可先安装http-server：

```bash
npm install -g http-server
```

启动该项目，默认为8080端口，可自定义端口号：

```bash
http-server –p 2017 swagger-editor
```

界面效果如下：

![](https://xnstatic-1253397658.file.myqcloud.com/swagger02.png)

写完文档后可导出yaml格式的文件，这个文件可在swagger ui中使用生成漂亮的HTML页面的API文档。

## swagger2markup

GitHub主页：https://github.com/Swagger2Markup/swagger2markup

swagger2markup用来将我们手写的或自动生成的swagger.yaml或swagger.json转换成漂亮的 AsciiDoc 或 Markdown格式文件。
然后再通过`asciidoctor-maven-plugin`将其转换为漂亮的HTML格式文档方便查看。

将swagger.yaml转换成AsciiDoc的方法：http://swagger2markup.github.io/swagger2markup/1.3.1/

先添加maven依赖：

```xml

<dependency>
    <groupId>io.github.swagger2markup</groupId>
    <artifactId>swagger2markup</artifactId>
    <version>1.3.1</version>
</dependency><!-- 日志打印 slf4j + log4j包 -->
<dependency>
<groupId>org.slf4j</groupId>
<artifactId>slf4j-log4j12</artifactId>
<version>1.7.25</version>
</dependency>
```

转换本地的yaml文件方法：

```java
import io.github.swagger2markup.GroupBy;
import io.github.swagger2markup.Language;
import io.github.swagger2markup.Swagger2MarkupConfig;
import io.github.swagger2markup.Swagger2MarkupConverter;
import io.github.swagger2markup.builder.Swagger2MarkupConfigBuilder;
import io.github.swagger2markup.markup.builder.MarkupLanguage;

import java.nio.file.Path;
import java.nio.file.Paths;

public static void main(String[] args) throws Exception {
    Path localSwaggerFile = Paths.get("src/main/resources/swagger.yaml");
    Path outputFile = Paths.get("build/swagger");

    Swagger2MarkupConfig config = new Swagger2MarkupConfigBuilder()
            .withMarkupLanguage(MarkupLanguage.ASCIIDOC)
            .withOutputLanguage(Language.ZH)
            .withPathsGroupedBy(GroupBy.TAGS)
            .withGeneratedExamples()
            .withoutInlineSchema()
            .build();
    Swagger2MarkupConverter converter = Swagger2MarkupConverter.from(localSwaggerFile)
            .withConfig(config)
            .build();
    converter.toFile(outputFile);
}
```

使用 [asciidoctor-maven-plugin](https://github.com/asciidoctor/asciidoctor-maven-plugin/blob/master/README_zh-CN.adoc)
将 AsciiDoc 转换成HTML/XML文件方法

添加maven插件：

```xml
<properties>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    <asciidoctor.maven.plugin.version>1.5.5</asciidoctor.maven.plugin.version>
    <asciidoctorj.version>1.5.5</asciidoctorj.version>
    <maven.build.timestamp.format>yyyy-MM-dd HH:mm:ss</maven.build.timestamp.format>
</properties>

<plugin>
    <groupId>org.asciidoctor</groupId>
    <artifactId>asciidoctor-maven-plugin</artifactId>
    <version>${asciidoctor.maven.plugin.version}</version>
    <configuration>
        <sourceDirectory>build</sourceDirectory>
        <outputDirectory>docs/asciidoc/${project.version}</outputDirectory>
        <headerFooter>true</headerFooter>
        <doctype>book</doctype>
        <sourceHighlighter>coderay</sourceHighlighter>
        <attributes>
            <toc>left</toc>
            <toclevels>3</toclevels>
            <sectnums>true</sectnums>
            <revnumber>${project.version}</revnumber>
            <revdate>${maven.build.timestamp}</revdate>
            <organization>广州恩智科技</organization>
            <sourcedir>${project.build.sourceDirectory}</sourcedir>
        </attributes>
    </configuration>
    <executions>
        <execution>
            <id>output-html</id>
            <phase>generate-resources</phase>
            <goals>
                <goal>process-asciidoc</goal>
            </goals>
            <configuration>
                <backend>html5</backend>
            </configuration>
        </execution>
        <execution>
            <id>output-docbook</id>
            <phase>generate-resources</phase>
            <goals>
                <goal>process-asciidoc</goal>
            </goals>
            <configuration>
                <backend>docbook</backend>
            </configuration>
        </execution>
    </executions>
</plugin>
```

然后执行：

```bash
mvn clean && mvn compile
```

那么在 `docs/asciidoc/1.0.0` 文件夹里面就会生成html、xml格式的文档，可以直接浏览器打开HTML文档。

## 生成PDF文档

使用`asciidoctor-maven-plugin`插件也能生成pdf格式的文档，但是对于中文支持太差了，很多中文字符是空白。

这里我通过另外的一种方式生成中文PDF文档。

参考：<https://github.com/chloerei/asciidoctor-pdf-cjk-kai_gen_gothic>

### 安装ruby

window上面直接去下载安装包：<https://rubyinstaller.org/downloads/>

更改国内源，参考 <https://gems.ruby-china.org/> ：

```bash
gem sources --add https://gems.ruby-china.org/ --remove https://rubygems.org/
gem sources -l
```

### 安装cjk依赖

```bash
gem install asciidoctor-pdf-cjk-kai_gen_gothic
```

### 下载中文字体

```bash
asciidoctor-pdf-cjk-kai_gen_gothic-install
```

执行这一步超时，解决办法是手动下载字体。

字体文件都在这里：<https://github.com/chloerei/asciidoctor-pdf-cjk-kai_gen_gothic/releases>

CN 主题需要的是 RobotoMono 开头和 KaiGenGothicCN 开头的字体文件，
尝试用其他工具手工下载到 gem 安装目录的 data/fonts 文件夹内。

查看gem安装目录命令：

```bash
gem environment
```

在返回结果里面找到这句：

```
INSTALLATION DIRECTORY: E:/Ruby24-x64/lib/ruby/gems/2.4.0
```

然后在这个目录下面进入目录 `gems/asciidoctor-pdf-cjk-kai_gen_gothic-0.1.1/data/fonts`，
将下载的字体都放进去。

### 使用方法

首先在`build/swagger.adoc`的顶部加入：

```
:toclevels: 3
:numbered:

```

注意有个空行分割，目的是左边导航菜单是3级，并且自动加序号。

为了美化显示，还要将`swagger.adoc`中全局替换一下，将

```
cols=".^2,.^3,.^9,.^4,.^2"
```

替换成：

```
cols=".^2,.^3,.^6,.^4,.^5"
```

然后执行：

```bash
asciidoctor-pdf -r asciidoctor-pdf-cjk-kai_gen_gothic -a pdf-style=KaiGenGothicCN build/swagger.adoc
```

会在`swagger.adoc`的同级目录生成`swagger.pdf`文件。

![](https://xnstatic-1253397658.file.myqcloud.com/swagger04.png)

## API开发规约

1. 接口维护者使用swagger2提供的swagger editor，依据OpenAPI规范手动编写swagger.yaml接口定义文档。
2. 生成在线的 HTML 格式的 API文档托管到网站上面，前端和后端工程师可在线查看api文档。
3. 如果有需要也生成一份PDF格式的文档，以便随时可以分发出去。
4. 如果开发过程中接口变更，接口维护者重新编写接口定义文档，然后重新生成新的API文档，并通知前端和后端工程师更新内容。

