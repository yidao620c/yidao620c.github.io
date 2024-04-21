---
title: CAS教程 - Restful API方式实现SSO
date: 2019-01-15 12:25:39 +0800
toc: true
categories: [网络安全]
tags: [ CAS ]
---

如果每个Web应用有自己的登录界面，不想使用Cas服务器的登录界面。也不需要改动Cas服务器去适配各种客户端登录界面需求。
那么可以通过使用Restful API的方式进行SSO认证，这是最佳实践。
<!-- more -->

## 系统架构

统一使用 SpringBoot+Meven，app1 和 app2 内容一致。

 文件名             | 域名                             | 功能                 
-----------------|--------------------------------|--------------------
 cas-app1        | http://app1.com:8181/fire      | 客户端1[cas-client在此] 
 cas-app2        | http://app2.com:8282/water     | 客户端2[cas-client在此] 
 sso-server      | http://sso.server.com:8889/sso | 自定义SSO服务，管理单点登录的用户 
 cas-server-rest | http://cas.server.com:8484/cas | CAS-Server         

本地hosts文件配置如下:

```
127.0.0.1    cas.server.com
127.0.0.1    sso.server.com
127.0.0.1    app1.com
127.0.0.1    app2.com
```

其实是将cas-server中的TGT管理，单独使用自定义的sso-server进行管理了，这种架构推荐的最佳方式。

![](https://xnstatic-1253397658.file.myqcloud.com/cas20190317-01.jpg)

## 操作流程

1. 用户访问受限资源 http://app1.com:8181/fire/books.html
1.
由于未登录，跳转自定义的登录界面 http://app1.com:8181/fire/login.html?service=http%3A%2F%2Fapp1.com%3A8181%2Ffire%2Fbooks.html
1. 登录页面首先向 sso-server 发起 jsonp 的登录验证请求（提交sso-server域名下的cookie），根据 Cookie 判断是否登录过

    未登录返回：jQuery33109023589333109241_1524470965614({"status":0,"data":"nothing"})
    登录过返回：jQuery33109023589333109241_1524470965614({"status":1,"data":"http://app1.com:8181/fire/books.html?ticket=ST值"})

（1）未登录

1. 未登录，渲染登录表单，用户进行username、password、service 登录，登录地址提交到 sso-server 的 login 接口
1. login 接口通过账号密码调用 cas-server 服务的 v1/tickets 接口，获取 TGT，然后根据用户名规则，将 TGT 保存到 Cookie 和
   Redis 缓存中
1. login 接口通过 TGT 获取 ST，拼接成 http://app1.com:8181/fire/books.html?ticket=ST值，进行redirect 该字符串作为响应
1. http://app1.com:8181/fire/books.html?ticket=ST值由 app 中的 cas-client 过滤器进行验证 ST 的正确性
1. 验证通过，建立成功的SessionId

（2）已登录

1. 已登录，接收 jsonp 响应的值 http://app1.com:8181/fire/books.html?ticket=ST值，并进行 JS 的重定向
1. http://app1.com:8181/fire/books.html?ticket=ST值  由 app 中的 cas-client 过滤器进行验证 ST 的正确性
1. 验证通过，建立成功的SessionId

## 实战

### cas-server 配置

如果需要使用rest的请求方式，就需要添加下面的依赖。

```xml
<!--开启cas server的rest支持-->
<dependency>
    <groupId>org.apereo.cas</groupId>
    <artifactId>cas-server-support-rest</artifactId>
    <version>${cas.version}</version>
</dependency>
```

### sso-server 配置

一些常规依赖，主要是redis

```xml
<!--HttpClient-->
<dependency>
   <groupId>org.apache.httpcomponents</groupId>
   <artifactId>httpclient</artifactId>
   <version>4.5.3</version>
</dependency>

<!--Gson-->
<dependency>
   <groupId>com.google.code.gson</groupId>
   <artifactId>gson</artifactId>
   <version>2.8.2</version>
</dependency>

<!--redis-->
<dependency>
   <groupId>org.springframework.boot</groupId>
   <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```

### cas-app配置

主要改动了一下login.html登录页面。增加一个拦截器判断用户是否登录

```js
$(document).ready(function() {
    var service = GetQueryString("service");
    // 如果为空，表示直接进入登录页面
    if (service == null) {
        service = "http://app1.com:8181/fire/index.html";
    }
    console.info("service：" + service);
    $("#service").val(service); // 受访受限url的地址
    // 新进行判断用户是否登录过
    $.ajax({
        method: "GET",
        url: "http://sso.server.com:9000/sso/user/check",
        data: {
            'service': service
        },
        xhrFields: {
            withCredentials: true
        },
        crossDomain: true,
        dataType: "jsonp",
        jsonp: "callback",
        // cache: false,
        success: function(result) {
            console.info("请求成功");
            console.info(result);
            if (result.status == 1) {
                // 设置 302 重定向跳转
                window.location.href = result.data;
            } else {
                // 显示登录页面
                $("#loginDiv").show("slow");
            }
        },
        error: function(data) {
            console.info("请求失败");
            $("#loginDiv").show("slow");
        }
    });
});
</script>
```

登录form提交的URL是 sso server的地址：

```html
<div id="loginDiv" style="display: none">
    <form action="http://sso.server.com:9000/sso/user/login" method="post">
        <table>
            <tr>
                <td>用户名:</td>
                <td><input id="username" name="username" type="text" ></td>
            </tr>
            <tr>
                <td>密 码:</td>
                <td><input id="password" name="password" type="password"></td>
            </tr>
            <tr>
                <td>
                    <input type="hidden" name="service" id="service" value="">
                    <input type="submit" value="登录">
                </td>
                <td><input type="reset"></td>
            </tr>
        </table>
    </form>
</div>
```

其他直接看github源码

