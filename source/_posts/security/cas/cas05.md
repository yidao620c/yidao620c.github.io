---
title: CAS教程 - Service配置
date: 2019-01-12 23:10:22 +0800
toc: true
categories: [网络安全]
tags: [ CAS ]
---

cas客户端接入称之为service，必须经过cas的允许才能进行登录，当然不同的客户端可以做不同的事情，其中包括：

1. 自定义主题（各客户端登录页自定义）
1. 自定义属性（服务属性（固定）与用户属性（动态））
1. 自定义协议
1. 自定义登录后跳转方式，跳转路径
1. 授权策略（拒绝属性、可登录时间范围限制、等等）
1. 拒绝授权模式

<!-- more -->

持久化策略：

* InMemory XML(通过spring bean进行内存存储)
* JSON(通过json文件存储)
* YAML(通过yml文件存储)
* Mongo(文档数据库持久化)
* JPA(关系型数据库持久化)
* DynameDb
* LDAP
* Cochbase

在sso初步上线时推荐采用json文件存储，后面逐步多服务注入时推荐采用Mongo进行存储，
采用cas-management 通过UI方式进行管理我们的数据，目前阶段，持久化策略必须和cas进行配置一致才能生效。

本章进行service的json配置及介绍，yml文件格式配置，参考文档：

https://apereo.github.io/cas/5.3.x/installation/JSON-Service-Management.html

接下来正式进入Service配置实战篇。

## JSON配置

这里我只允许http://localhost开头请求的service才能认证。

在resources/services下新建文件localhost-10000002.json，内容如下：

```json
{
  "@class": "org.apereo.cas.services.RegexRegisteredService",
  "serviceId": "^(http)://localhost.*",
  "name": "本地服务",
  "id": 10000002,
  "description": "这是一个本地允许的服务，通过localhost访问都允许通过",
  "evaluationOrder": 1
}
```

注意：json文件名字规则为`${name}-${id}.json`, id必须为json文件内容id一致。

json文件解释：

* @class：必须为org.apereo.cas.services.RegisteredService的实现类，
  对其他属性进行一个json反射对象，常用的有RegexRegisteredService，匹配策略为id的正则表达式
* serviceId：唯一的服务id
* name： 服务名称，会显示在默认登录页
* id：全局唯一标志
* description：服务描述，会显示在默认登录页
* evaluationOrder： 匹配争取时的执行循序（越小越优先）
* theme：主题，默认是apereo

除了以上说的还有很多配置策略以及节点，具体看官方文档, 配置不同的RegisteredService也会有稍微不一样

## 启用识别

上面新建了json文件cas还不知道要去识别json，需要打开开关.

```properties
#开启识别json文件，默认false
cas.serviceRegistry.initFromJson=true
#自动扫描服务配置，默认开启
#cas.serviceRegistry.watcherEnabled=true
#120秒扫描一遍
#cas.serviceRegistry.schedule.repeatInterval=120000
#延迟15秒开启
#cas.serviceRegistry.schedule.startDelay=15000
#默认json/yml资源加载路径为resources/services
#cas.serviceRegistry.json.location=classpath:/services
```

所有配置可以参考：

https://apereo.github.io/cas/5.3.x/installation/Configuration-Properties.html

另外还需要在 pom.xml 文件中加入依赖配置

```xml
<!--json服务注册-->
<dependency>
    <groupId>org.apereo.cas</groupId>
    <artifactId>cas-server-support-json-service-registry</artifactId>
    <version>${cas.version}</version>
</dependency>
```

## 自定义登录界面和主题

主题就意味着风格不一，目的就是为了在不同的接入端（service）展示不同的页面，
就例如淘宝登录、天猫登录，其中登录点还是一个sso，但淘宝登录卖的广告是淘宝的，而天猫登录卖的广告是天猫的。

简略看完后，会有以下的规范：

1. 静态资源(js,css)存放目录为src/main/resources/static
1. html资源存(thymeleaf)放目录为src/main/resources/templates
1. 主题配置文件存放在src/main/resources并且命名为[theme_name].properties
1. 主题页面html存放目录为src/main/resources/templates/<theme-id>

官方文档明确说明，登录页渲染文件为casLoginView.html,那意味我们在主题具体目录下新增改文件并且按照cas要求写那就可以了

### 接入服务指定主题

```json
{
  "@class" : "org.apereo.cas.services.RegexRegisteredService",
  "serviceId" : "^(http|https)://app1.*",
  "name" : "app1",
  "id" : 100006,
  "description": "这是一个app1域名下的服务，通过app1访问都允许通过",
  "evaluationOrder": 10,
  "theme" : "app1"
}
```

### 修改默认主题

```properties
cas.theme.defaultThemeName=app1
```

配置默认后，将直接指定该主题，比如直接访问cas登录，没有传递service参数时等。

### css样式表

`app1/css/main.css` 设置一个标题为粉色

h3 {
color: pink; /**粉色**/
}

### 主题配置文件

app1.properties

```properties
css.file=/themes/app1/css/main.css
pageTitle= 这是APP 1 网站

#standard.custom.css.file=/themes/[theme_name]/css/cas.css
#cas.javascript.file=/themes/[theme_name]/js/cas.js

standard.custom.css.file=/css/cas.css
admin.custom.css.file=/css/admin.css
cas.javascript.file=/js/cas.js
```

### 登录模版

```html
<!DOCTYPE html>
<html>

<head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1" />
    <meta http-equiv="X-UA-Compatible" content="IE=edge" />
    <title th:text="${#themes.code('pageTitle')}"></title>
    <link rel="stylesheet" th:href="@{${#themes.code('css.file')@@" />
</head>

<body>
<h3 th:text="${#themes.code('pageTitle')}"></h3>
<div>
    <form method="post" th:object="${credential}">
        <div th:if="${#fields.hasErrors('*')}"><span th:each="err : ${#fields.errors('*')}" th:utext="${err}" />
        </div>
        <h4 th:utext="#{screen.welcome.instructions}"></h4>
        <section class="row">
            <label for="username" th:utext="#{screen.welcome.label.netid}" />
            <div th:unless="${openIdLocalId}">
                <input class="required" id="username" size="25" tabindex="1" type="text" th:disabled="${guaEnabled}" th:field="*{username}" th:accesskey="#{screen.welcome.label.netid.accesskey}" autocomplete="off" th:value="casuser" />
            </div>
        </section>
        <section class="row">
            <label for="password" th:utext="#{screen.welcome.label.password}" />
            <div>
                <input class="required" type="password" id="password" size="25" tabindex="2" th:accesskey="#{screen.welcome.label.password.accesskey}" th:field="*{password}" autocomplete="off" th:value="Mellon" />
            </div>
        </section>
        <section>
            <input type="hidden" name="execution" th:value="${flowExecutionKey}" />
            <input type="hidden" name="_eventId" value="submit" />
            <input type="hidden" name="geolocation" />
            <input class="btn btn-submit btn-block" name="submit" accesskey="l" th:value="#{screen.welcome.button.login}" tabindex="6" type="submit" />
        </section>
    </form>
</div>
</body>

</html>
```

