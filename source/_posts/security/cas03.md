---
title: "CAS教程-定制开发"
date: 2019-01-05 21:15:34 +0800
comments: true
toc: true
categories: 网络安全
tags: [CAS]
---

现在开始对CasServer进行二次开发，比如如何设置数据库连接，如何使用数据库的用户名和密码登录，
如何使用Restful API方式实现SSO，如何自定义服务，如何自定义登陆界面等等。接下来将逐步介绍。

Cas官方说明，如果你想对它默认项目有所更改，那么就使用覆盖它路径的方式进行。<!--more-->

更改CAS的配置既可以修改cas.properties文件，也能修改默认的application.properties。
为了配置集中处理，我会把默认的application.properties配置文件提出来，只修改它，并把cas.properties内容都移到里面去。

## 修改默认用户名和密码

在cas-overlay-template-master项目中，新建一个src/main/resources目录，将resources目录设置成资源目录。
然后将`将target/war/work/org.apereo.cas/cas-server-webapp-tomcat/WEB-INF/classes/apereo.properties`复制到resources目录中。

将前面修改的`cas.properties`内容复制到里面去。

然后找到最后的`cas.authn.accept.users=casuser::Mellon`，改成我自己的：

```
cas.authn.accept.users=xiongneng::xiongneng
```

重新build后放到tomcat容器中，再看看效果。使用新的用户名密码登录成功！

