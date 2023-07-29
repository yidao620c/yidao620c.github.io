---
title: CAS教程 - 原理篇
date: 2019-01-02 09:12:33 +0800
toc: true
categories: [ 网络安全 ]
tags: [ CAS ]
---

所谓单点登录(SSO)，就是同平台的诸多应用登陆一次，下一次就免登陆的功能。就像你在知乎首页登录一次，
下一次再访问知乎专栏或是知乎日报就可以免去登录操作。

实现SSO的方式有很多，现在主流的就是CAS这种基于session的单点登录形式。
CAS是一个单点登录框架，开始是由耶鲁大学的一个组织开发，后来归到apereo去管。
CAS也是开源，遵循着apache 2.0协议，代码目前是在github上管理。

接下来我会写一个CAS的系列教程，这篇是一个原理篇，后面系列教程是实际操作篇。
<!-- more -->

## 结构

CAS分为两部分，CAS Server和CAS Client

* CAS Server用来负责用户的认证工作，就像是把第一次登录用户的一个标识存在这里，
  以便此用户在其他系统登录时验证其需不需要再次登录。
* CAS Client就是我们自己的应用，需要接入CAS Server端。当用户访问我们的应用时，
  首先需要重定向到CAS Server端进行验证，要是原来登陆过，就免去登录，重定向到下游系统，否则进行用户名密码登陆操作。

## 术语

* Ticket Granting ticket (TGT) ：可以认为是CAS Server根据用户名密码生成的一张票，存在server端
* Ticket-granting cookie (TGC) ：其实就是一个cookie，存放用户身份信息，由server发给client端
* Service ticket (ST) ：由TGT生成的一次性票据，用于验证，只能用一次。相当于server发给client一张票，
  然后client拿着这是个票再来找server验证，看看是不是server签发的。
* 全局会话 ：用户到认证中心登录后，用户和认证中心之间建立起了会话。
* 局部会话 ：在系统应用和用户浏览器之间建立的会话。

## 登录流程图

![](https://xnstatic-1253397658.file.myqcloud.com/cas20190218-01.jpg)

1)、问：系统A是如何发现该请求需要登录重定向到认证中心的？

答：用户通过浏览器地址栏访问系统A，系统A(也可以称为CAS客户端)去Cookie中拿JSESSION，
即在Cookie中维护的当前回话session的id，如果拿到了，说明用户已经登录，如果未拿到，说明用户未登录。

2)、问：系统A重定向到认证中心，发送了什么信息或者地址变成了什么？

答：假如系统A的地址为http://a:8080/，CAS认证中心的服务地址为http://cas.server:8080/，
那么重点向前后地址变化为：http://a:8080/ ==> http://cas.server:8080/?service=http://a:8080/，
由此可知，重点向到认证中心，认证中心拿到了当前访问客户端的地址。

3)、问：登录成功后，认证中心重定向请求到系统A，认证通过令牌是如何附加发送给系统A的？

答：重定向之后的地址栏变成：http://a:8080/?ticket=ST-XXXX-XXX，将票据以ticket为参数名的方式通过地址栏发送给系统A

4)、问：系统A验证令牌，怎样操作证明用户登录的？

答：系统A通过地址栏获取ticket的参数值ST票据，然后从后台将ST发送给CAS server认证中心验证，验证ST有效后，
CAS server返回当前用户登录的相关信息，系统A接收到返回的用户信息，并为该用户创建session会话，会话id由cookie维护，来证明其已登录。

5)、问：登录B系统，认证中心是如何判断用户已经登录的？

答：在系统A登录成功后，用户和认证中心之间建立起了全局会话，这个全局会话就是TGT(Ticket Granting Ticket)，
TGT位于CAS服务器端，TGT并没有放在Session中，也就是说，CAS全局会话的实现并没有直接使用Session机制，
而是利用了Cookie自己实现的，这个Cookie叫做TGC(Ticket Granting Cookie)，它存放了TGT的id,保存在用户浏览器上。

浏览器访问另一应用B需登录受限资源，此时进行登录检查，发现未登录，然后进行获取临时票据（ST）操作，发现没有临时票据（ST）。

系统B发现该请求需要登录，将请求重定向到CAS认证中心，获取全局票据TGT操作，获取全局票据，可以获得，认证中心发现已经登录。

CAS认证中心发放临时票据(ST)，并携带该票据ST重定向到系统B。

此时系统B再次进行登录检查，发现未登录，然后再次获取票据操作，此时可以获得票据(令牌)，
系统B再去cas-server验证一下是否为自己签发的，验证通过后就会返回用户信息，并建立局部会话。

6)、问：登出的过程，各个系统对当前用户都做了什么？

答：认证中心清除当前用户的全局会话TGT，同时清掉cookie中TGT的id：TGC；
然后是各个客户端系统，比如系统A、系统B，清除局部会话session，同时清掉cookie中session会话id：jsession

参考：

* <https://zhuanlan.zhihu.com/p/25007591>
* <https://blog.csdn.net/javaloveiphone/article/details/52439613>
