---
layout: post
title: SiteMesh使用笔记
date: 2015-03-28 10:42:23 +0800
comments: true
toc: true
categories: java
tags:
  - sitemesh
abbrlink: 36004
---

SiteMesh的介绍就不多说了，主要是用来统一页面风格，减少重复编码的。

它定义了一个过滤器，然后把页面都加上统一的头部和底部。

在maven工程的pom.xml中加入SiteMesh依赖：

```xml
<dependency>
    <groupId>opensymphony</groupId>
    <artifactId>sitemesh</artifactId>
    <version>2.4.2</version>
</dependency>
```
<!-- more -->

### 过滤器定义：
在web.xml中：

```xml
<filter>
    <filter-name>sitemeshFilter</filter-name>
    <filter-class>com.opensymphony.sitemesh.webapp.SiteMeshFilter</filter-class>
</filter>
<filter-mapping>
    <filter-name>sitemeshFilter</filter-name>
    <url-pattern>*.html</url-pattern>
</filter-mapping>
```

### WEB-INF下新建decorators.xml文件

```xml
<?xml version="1.0" encoding="UTF-8"?>
<decorators defaultdir="/WEB-INF/layouts/">
    <!-- 此处用来定义不需要过滤的页面 -->
    <excludes>
        <pattern>/static/*</pattern>
    </excludes>
    <!-- 用来定义装饰器要过滤的页面 -->
    <decorator name="default" page="default.jsp">
        <pattern>/*</pattern>
    </decorator>
</decorators>
```

不用过滤/static/目录下的文件，然后指定了装饰器：/WEB-INF/layouts/default.jsp。

### default.jsp模板
```jsp
<%@ page contentType="text/html;charset=UTF-8" %>
<%@ taglib prefix="sitemesh" uri="http://www.opensymphony.com/sitemesh/decorator" %>
<%@ taglib prefix="c" uri="http://java.sun.com/jsp/jstl/core" %>
<c:set var="ctx" value="${pageContext.request.contextPath}"/>
<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 4.01 Transitional//EN">
<html>
<head>
    <title>快贷发平台:<sitemesh:title/></title>
    <meta http-equiv="Content-Type" content="text/html;charset=utf-8"/>
    <meta name="viewport" content="width=device-width, initial-scale=1, maximum-scale=1, user-scalable=no"/>

    <link type="image/x-icon" href="${ctx}/static/images/favicon.ico" rel="shortcut icon">
    <link href="${ctx}/static/bootstrap/3.3.1/css/bootstrap.min.css" type="text/css" rel="stylesheet"/>
    <link href="${ctx}/static/jquery-validation/1.11.1/validate.css" type="text/css" rel="stylesheet"/>
    <link href="${ctx}/static/styles/default.css" type="text/css" rel="stylesheet"/>
    <script src="${ctx}/static/jquery/jquery-1.9.1.min.js" type="text/javascript"></script>
    <script src="${ctx}/static/jquery-validation/1.11.1/jquery.validate.min.js" type="text/javascript"></script>
    <script src="${ctx}/static/jquery-validation/1.11.1/messages_bs_zh.js" type="text/javascript"></script>
    <sitemesh:head/>
</head>

<body>
<div class="container">
    <%@ include file="/WEB-INF/layouts/header.jsp" %>
    <div id="content">
        <sitemesh:body/>
    </div>
    <%@ include file="/WEB-INF/layouts/footer.jsp" %>
</div>
<script src="${ctx}/static/bootstrap/3.3.1/js/bootstrap.min.js" type="text/javascript"></script>
</body>
</html>
```

首先引入了SiteMesh标签

* `<sitemesh:title/>`会自动替换为被过滤页面的title
* `<sitemesh:head/>`会把被过滤页面head里面的东西（除了title）放在这个地方
* `<sitemesh:body/>`会把被过滤的页面body里面的内容放在这里
* 在content的上下引入了header和footer，我们在头部引入了js和css，就可以重用了

使用的过程中，几乎感受不到SiteMesh的存在，例如下面的页面：

```jsp
<%@ page language="java" contentType="text/html; charset=utf-8"
    pageEncoding="utf-8"%>
<!DOCTYPE html PUBLIC "-//W3C//DTD HTML 4.01 Transitional//EN" "http://www.w3.org/TR/html4/loose.dtd">
<html>
<!-- 第一个被装饰(目标)页面  -->
<head>
<meta http-equiv="Content-Type" content="text/html; charset=utf-8">
<title>被装饰（目标）页面title</title>
<script type="text/javascript" src="/js/hello.js"></script>
</head>

<body>
    <h4>被装饰（目标）页面body标签内内容。</h4>
    <h3>使用SiteMesh的好处?</h3>
    <ul>
        <li>被装饰（目标）页面和装饰页面完全分离。</li>
        <li>做到真正的页面复用，一个装饰页面装饰多个被装饰（目标）页面。</li>
        <li>更容易实现统一的网站风格。</li>
        <li>还有。。。</li>
    </ul>
</body>
</html>
```

这就是一个普通的页面，但是被SiteMesh装饰之后，就会自动去掉`<html> <body> <head>`等元素，
然后把相应的东西放在模板对应位置上。

我们来看一下，被SiteMesh装饰过的页面源代码：

```html
<!DOCTYPE html>
<html>
<head>
<title>QuickStart示例:被装饰（目标）页面title</title>
<meta http-equiv="Content-Type" content="text/html;charset=utf-8" />
<meta http-equiv="Cache-Control" content="no-store" />
<meta http-equiv="Pragma" content="no-cache" />
<meta http-equiv="Expires" content="0" />

<link type="image/x-icon" href="/SpringMVC/static/images/favicon.ico" rel="shortcut icon">
<link href="/SpringMVC/sc/bootstrap/2.3.0/css/bootstrap.min.css" type="text/css" rel="stylesheet" />
<link href="/SpringMVC/sc/jquery-validation/1.11.0/validate.css" type="text/css" rel="stylesheet" />
<link href="/SpringMVC/css/base/default.css" type="text/css" rel="stylesheet" />
<script src="/SpringMVC/sc/jquery/jquery-1.9.1.min.js" type="text/javascript"></script>
<script src="/SpringMVC/sc/jquery-validation/1.11.0/jquery.validate.min.js" type="text/javascript"></script>
<script src="/SpringMVC/sc/jquery-validation/1.11.0/messages_bs_zh.js" type="text/javascript"></script>

<meta http-equiv="Content-Type" content="text/html; charset=utf-8">
<script type="text/javascript" src="/js/hello.js"></script>
</head>
<body>
    <div class="container">
<div id="header">
</div>
<div id="content">
    <h4>被装饰（目标）页面body标签内内容。</h4>
    <h3>使用SiteMesh的好处?</h3>
    <ul>
        <li>被装饰（目标）页面和装饰页面完全分离。</li>
        <li>做到真正的页面复用，一个装饰页面装饰多个被装饰（目标）页面。</li>
        <li>更容易实现统一的网站风格。</li>
        <li>还有。。。</li>
    </ul>
</div>
<div id="footer">
    Copyright &copy; 2005-2012 <a href="">spring.org.cn</a>
</div>
</div>
<script src="/SpringMVC/sc/bootstrap/2.3.0/js/bootstrap.min.js" type="text/javascript"></script>
</body>
</html>
```

### 总结
SiteMesh配置简单，使用灵活，几乎不会影响我们的正常开发，作为一个默默无闻的好帮手，非常推荐使用。

