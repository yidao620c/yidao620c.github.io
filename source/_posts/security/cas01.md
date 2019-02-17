---
title: "CAS实现单点登录原理"
date: 2019-01-02 09:12:33 +0800
comments: true
toc: true
categories: 网络安全
tags: [CAS]
---

所谓单点登录(SSO)，就是同平台的诸多应用登陆一次，下一次就免登陆的功能。就像你在知乎首页登录一次，
下一次再访问知乎专栏或是知乎日报就可以免去登录操作。

实现SSO的方式有很多，现在主流的就是CAS这种基于session的单点登录形式。
CAS是一个单点登录框架，开始是由耶鲁大学的一个组织开发，后来归到apereo去管。 
CAS也是开源，遵循着apache 2.0协议，代码目前是在github上管理。

接下来我会写一个CAS的系列教程，这篇是一个原理篇，后面系列教程是实际操作篇。<!--more-->

参考：

* https://zhuanlan.zhihu.com/p/25007591
* https://blog.csdn.net/javaloveiphone/article/details/52439613
