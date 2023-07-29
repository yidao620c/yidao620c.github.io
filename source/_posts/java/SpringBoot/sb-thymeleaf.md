---
title: SpringBoot系列 - 集成Thymeleaf构建Web应用
date: 2017-07-01 18:17:29 +0800
toc: true
categories: [ java ]
tags: [ spring ]
---

Thymeleaf 是一个跟 Velocity、FreeMarker 类似的模板引擎，它可以完全替代 JSP 。相较与其他的模板引擎，它有如下三个极吸引人的特点：

1. Thymeleaf 在有网络和无网络的环境下皆可运行，即它可以让美工在浏览器查看页面的静态效果，也可以让程序员在服务器查看带数据的动态页面效果。
2. Thymeleaf 开箱即用的特性。它提供标准和spring标准两种方言，可以直接套用模板实现JSTL、 OGNL表达式效果，避免每天套模板、该jstl、改标签的困扰。
3. Thymeleaf 提供spring标准方言和一个与 SpringMVC 完美集成的可选模块，可以快速的实现表单绑定、属性编辑器、国际化等功能。

目前最新版本是Thymeleaf 3，官网地址：<http://www.thymeleaf.org>

本篇将介绍如何在SpringBoot中集成Thymeleaf构建Web应用。
<!-- more -->

## maven依赖

```xml

<properties>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
    <java.version>1.8</java.version>
    <thymeleaf.version>3.0.7.RELEASE</thymeleaf.version>
    <thymeleaf-layout-dialect.version>2.2.2</thymeleaf-layout-dialect.version>
</properties>

<dependencies>

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
    <exclusions>
        <exclusion>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-tomcat</artifactId>
        </exclusion>
    </exclusions>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-jetty</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-thymeleaf</artifactId>
</dependency>
</dependencies>
```

## 资源目录

Spring Boot默认提供静态资源目录位置需置于classpath下，目录名需符合如下规则：

1. /static
1. /public
1. /resources
1. /META-INF/resources

举例：我们可以在`src/main/resources/`目录下创建static，在该位置放置一个图片文件`pic.jpg`。
启动程序后，尝试访问`http://localhost:8080/pic.jpg`。如能显示图片，配置成功。

SpringBoot的默认模板路径为：`src/main/resources/templates`

## 添加页面

这里我在`src/main/resources/templates`下面添加一个`index.html`首页，同时引用到一些css、js文件，看看Thymeleaf 3的语法：

```html
<!DOCTYPE html>
<html xmlns="http://www.w3.org/1999/xhtml"
      xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <meta name="renderer" content="webkit">
    <title>首页</title>
    <link rel="shortcut icon" th:href="@{/favicon.ico}"/>
    <link th:href="@{/static/css/bootstrap.min.css}" rel="stylesheet"/>
    <link th:href="@{/static/css/font-awesome.min.css}" rel="stylesheet"/>
</head>
<body class="fixed-sidebar full-height-layout gray-bg" style="overflow:hidden">
<div id="wrapper">
    <h1 th:text="${msg}"></h1>
</div>
<script th:src="@{/static/js/jquery.min.js}"></script>
<script th:src="@{/static/js/bootstrap.min.js}"></script>

</body>
</html>
```

页面中几个地方说明一下：

```
<html xmlns="http://www.w3.org/1999/xhtml"
      xmlns:th="http://www.thymeleaf.org">
```

加上这个命名空间后就能在页面中使用Thymeleaf自己的标签了，以`th:`开头。

连接语法 `th:href="@{/static/css/bootstrap.min.css}"`

访问Model中的数据语法 `th:text="${msg}"`

这里我在页面里显示的是一个键为`msg`的消息，需要后台传递过来。

Thymeleaf 3的详细语法规则请参考 [官网教程](http://www.thymeleaf.org/doc/tutorials/3.0/usingthymeleaf.html)

## 编写Controller

接下来编写控制器类，将URL `/` 和 `/index` 都返回index.html页面：

```java
@Controller
public class IndexController {
    private static final Logger _logger = LoggerFactory.getLogger(IndexController.class);
    /**
     * 主页
     *
     * @param model
     * @return
     */
    @RequestMapping({"/", "/index"})
    public String index(Model model) {
        model.addAttribute("msg", "welcome you!");
        return "index";
    }
}
```

这里，我再model里面添加了一个属性，key=`msg`，值为`welcome you!`。

接下来启动应用后，打开首页看看效果如何。消息正确显示，成功！。

## GitHub源码

[springboot-thymeleaf](https://github.com/yidao620c/SpringBootBucket/tree/master/springboot-thymeleaf)

