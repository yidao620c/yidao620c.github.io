---
title: nginx反向代理WebSocket
date: 2018-03-12 11:24:23 +0800
toc: true
categories: [ 中间件 ]
tags: [ nginx ]
---

WebSocket协议相比较于HTTP协议成功握手后可以多次进行通讯，直到连接被关闭。但是WebSocket中的握手和HTTP中的握手兼容，
它使用HTTP中的Upgrade协议头将连接从HTTP升级到WebSocket。这使得WebSocket程序可以更容易的使用现已存在的基础设施。

WebSocket工作在HTTP的80和443端口并使用前缀ws://或者wss://进行协议标注，在建立连接时使用HTTP/1.1的101状态码进行协议切换，
当前标准不支持两个客户端之间不借助HTTP直接建立Websocket连接。

更多Websocket的介绍可参考我写的 [聊一聊WebSocket](https://www.xncoding.com/2017/05/03/web/websocket.html) 一文。

开发小程序的时候需要用到WebSocket长连接和推送技术，但是必须使用wss，并且必须通过域名访问。这时候就需要用到nginx反向代理了。
<!-- more -->

## 原理

一般我们开发的WebSocket服务程序使用ws协议，明文的。但是怎样让它安全的通过互联网传输呢？这时候可以通过nginx在客户端和服务端直接做一个转发了，
客户端通过wss访问，然后nginx和服务端通过ws协议通信。如下图所示：

![](https://xnstatic-1253397658.file.myqcloud.com/nginx-websocket01.png)

## 配置

前提条件是你有一个域名，并且申请好了证书。

新建nginx配置文件`/etc/nginx/conf.d/websocket.conf`，内容如下：

```nginx
map $http_upgrade $connection_upgrade {
    default upgrade;
    '' close;
}

upstream websocket {
    server localhost:8282; # appserver_ip:ws_port
}

server {
     server_name test.enzhico.net;
     listen 443 ssl;
     location / {
         proxy_pass http://websocket;
         proxy_read_timeout 300s;
         proxy_send_timeout 300s;
         
         proxy_set_header Host $host;
         proxy_set_header X-Real-IP $remote_addr;
         proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
         
         proxy_http_version 1.1;
         proxy_set_header Upgrade $http_upgrade;
         proxy_set_header Connection $connection_upgrade;
     }
    ssl_certificate /etc/letsencrypt/live/test.enzhico.net/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/test.enzhico.net/privkey.pem;
}
```

之类解释一下关键配置部分：

最重要的就是在反向代理的配置中增加了如下两行，其它的部分和普通的HTTP反向代理没有任何差别。

```nginx
proxy_set_header Upgrade $http_upgrade;
proxy_set_header Connection $connection_upgrade;
```

这里面的关键部分在于HTTP的请求中多了如下头部：

```
Upgrade: websocket
Connection: Upgrade
```

这两个字段表示请求服务器升级协议为WebSocket。服务器处理完请求后，响应如下报文：

```
# 状态码为101
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: upgrade
```

告诉客户端已成功切换协议，升级为Websocket协议。握手成功之后，服务器端和客户端便角色对等，就像普通的Socket一样，能够双向通信。
不再进行HTTP的交互，而是开始WebSocket的数据帧协议实现数据交换。

这里使用map指令可以将变量组合成为新的变量，会根据客户端传来的连接中是否带有Upgrade头来决定是否给源站传递Connection头，
这样做的方法比直接全部传递upgrade更加优雅。

默认情况下，连接将会在无数据传输60秒后关闭，`proxy_read_timeout`参数可以延长这个时间。源站通过定期发送ping帧以保持连接并确认连接是否还在使用。

## 两个超时参数

**proxy_read_timeout**

> 语法 proxy_read_timeout time
> 默认值 60s
> 上下文 http server location
> 说明 该指令设置与代理服务器的读超时时间。它决定了nginx会等待多长时间来获得请求的响应。
> 这个时间不是获得整个response的时间，而是两次reading操作的时间。

**proxy_send_timeout**

> 语法 proxy_send_timeout time
> 默认值 60s
> 上下文 http server location
> 说明 这个指定设置了发送请求给upstream服务器的超时时间。超时设置不是为了整个发送期间，而是在两次write操作期间。
> 如果超时后，upstream没有收到新的数据，nginx会关闭连接

## 多次代理转发

工作中遇见过一种情况，就是某个域名在移动网络下面访问不了，这样的话我需要通过一个前段代理服务器做转发，这样就涉及到两次代理。

比如访问的websocket服务URL为：

```
wss://test.enzhico.net
```

这个在腾讯云公网IP上面，所有网络都能访问。另外一个域名`board.xncoding.com`解析到电信网络，部署在网关中心，这个域名腾讯云可以访问到。

在腾讯云主机上面：

```nginx
map $http_upgrade $connection_upgrade {
    default upgrade;
    '' close;
}

server {
     server_name test.enzhico.net;
     location / {
         proxy_pass http://board.xncoding.com;
         proxy_read_timeout 300s;
         proxy_send_timeout 300s;
         #proxy_set_header Host $host;
         proxy_set_header X-Real-IP $remote_addr;
         proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
         proxy_http_version 1.1;
         proxy_set_header Upgrade $http_upgrade;
         proxy_set_header Connection $connection_upgrade;
    }
    listen 443 ssl; # managed by Certbot
    ssl_certificate /etc/letsencrypt/live/test.enzhico.net/fullchain.pem; # managed by Certbot
    ssl_certificate_key /etc/letsencrypt/live/test.enzhico.net/privkey.pem; # managed by Certbot
    include /etc/letsencrypt/options-ssl-nginx.conf; # managed by Certbot
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem; # managed by Certbot
}

```

上面唯一要注意的是忙，把`proxy_set_header Host $host;`这一行注释掉了。

而在网关中心主机上面：

```nginx
map $http_upgrade $connection_upgrade {
    default upgrade;
    '' close;
}

upstream websocket {
    server localhost:8282; # appserver_ip:ws_port
}

server {
    listen 80;
    server_name board.xncoding.com;
    location / {
        proxy_pass http://websocket;
        proxy_read_timeout 300s;
        proxy_send_timeout 300s;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $connection_upgrade;
    }
}
```

只需要最外层使用wss协议，里面的交互都使用ws协议，所以监听80端口即可。

## 参考

* [Websockets SSL/TLS Termination Using NGINX Proxy](http://pankajmalhotra.com/Websockets-SSL-TLS-Termination-Using-NGINX-Proxy)
* [配置Nginx反向代理WebSocket](https://www.hi-linux.com/posts/42176.html)

