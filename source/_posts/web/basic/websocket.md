---
title: 聊一聊WebSocket
date: 2017-05-03 12:09:22 +0800
toc: true
categories: [ web ]
tags: [ web ]
---

WebSocket是HTML5开始提供的一种浏览器与服务器间进行全双工通讯的网络技术。
依靠这种技术可以实现客户端和服务器端的长连接，双向实时通信。

它的最大特点就是，服务器可以主动向客户端推送信息，客户端也可以主动向服务器发送信息，
是真正的双向平等对话，属于服务器推送技术的一种。

其他特点包括：

1. 建立在 TCP 协议之上，服务器端的实现比较容易。
2. 与 HTTP 协议有着良好的兼容性。默认端口也是80和443，并且握手阶段采用 HTTP 协议，因此握手时不容易屏蔽，能通过各种 HTTP
   代理服务器。
3. 数据格式比较轻量，性能开销小，通信高效。
4. 可以发送文本，也可以发送二进制数据。
5. 没有同源限制，客户端可以与任意服务器通信。
6. 协议标识符是ws（如果加密，则为wss），服务器网址就是 URL。

协议标识符是ws（如果加密，则为wss），服务器网址就是 URL

```
ws://xncoding.com:80/some/path
```

另外客户端不只是浏览器，只要实现了ws或者wss协议的客户端socket都可以和服务器进行通信。
<!-- more -->

## websocket协议

Websocket其实是一个新协议，借用了HTTP的协议来完成一部分握手，只是为了兼容现有浏览器的握手规范而已。

一个典型的Websocket握手（借用Wikipedia）

```
GET /chat HTTP/1.1
Host: server.example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: x3JJHMbDL1EzLkh9GBhXDw==
Sec-WebSocket-Protocol: chat, superchat
Sec-WebSocket-Version: 13
Origin: http://example.com
```

其中的

```
Upgrade: websocket
Connection: Upgrade
```

这个就是Websocket的核心了，告诉Apache、Nginx等服务器：
注意啦，我发起的是Websocket协议，快点帮我找到对应的助理处理~不是那个老土的HTTP。

服务器返回：

```
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: HSmrc0sMlYUkAGmm5OPpG2HaGWk=
Sec-WebSocket-Protocol: chat
```

至此，HTTP已经完成它所有工作了，接下来就是完全按照Websocket协议进行了。

## TCP长连接与超时

### 什么是长连接

先说短连接, 短连接是通讯双方有数据交互时就建立一个连接, 数据发送完成后，则断开此连接。

长连接就是大家建立连接之后, 不主动断开. 双方互相发送数据, 发完了也不主动断开连接,
之后有需要发送的数据就继续通过这个连接发送。

![](https://xnstatic-1253397658.file.myqcloud.com/websocket01.png)

TCP连接在默认的情况下就是所谓的长连接, 也就是说连接双方都不主动关闭连接, 这个连接就应该一直存在。

但是网络中的情况是复杂的, 这个连接可能会被切断. 比如客户端到服务器的链路因为故障断了, 或者服务器宕机了,
或者是你家网线被人剪了, 这些都是一些莫名其妙的导致连接被切断的因素, 还有几种比较特殊的。

### NAT超时

因为IPv4地址不足, 或者我们想通过无线路由器上网, 我们的设备可能会处在一个NAT设备的后面, 生活中最常见的NAT设备是家用路由器。

NAT设备会在IP封包通过设备时修改源/目的IP地址。对于家用路由器来说, 使用的是网络地址端口转换(NAPT),
它不仅改IP, 还修改TCP和UDP协议的端口号, 这样就能让内网中的设备共用同一个外网IP。举个例子, NAPT维护一个类似下表的NAT表：

![](https://xnstatic-1253397658.file.myqcloud.com/websocket02.png)

NAT设备会根据NAT表对出去和进来的数据做修改, 比如将`192.168.0.3:8888`发出去的封包改成`120.132.92.21:9202`,
外部就认为他们是在和`120.132.92.21:9202`通信。同时NAT设备会将`120.132.92.21:9202`
收到的封包的IP和端口改成`192.168.0.3:8888`,
再发给内网的主机, 这样内部和外部就能双向通信了, 但如果其中`192.168.0.3:8888 == 120.132.92.21:9202`
这一映射因为某些原因被NAT设备淘汰了, 那么外部设备就无法直接与`192.168.0.3:8888`通信了。

我们的设备经常是处在NAT设备的后面, 比如在大学里的校园网, 查一下自己分配到的IP, 其实是内网IP, 表明我们在NAT设备后面,
如果我们在寝室再接个路由器, 那么我们发出的数据包会多经过一次NAT。

国内移动无线网络运营商在链路上一段时间内没有数据通讯后, 会淘汰NAT表中的对应项, 造成链路中断。

### 网络状态切换

手机网络和WIFI网络切换, 网络断开和连上等情况, 也会使长连接断开. 这里原因可能比较多, 但结果无非就是IP变了, 或者被系统通知连接断了。

## 心跳包的作用

网上很多文章介绍长连接的时候都说：

* 因为是长连接, 所以需要定期发送心跳包。
* 心跳包是用来通知服务器客户端当前状态。

提出这些说法的人其实自己也是一知半解. 这些说法其实都对, 但是没有答到点上。

明确一点, TCP长连接本质上不需要心跳包来维持, 大家可以试一试, 让两台电脑连上同一个wifi, 然后让其中一台做服务器,
另一台用一个普通的没有设置KeepAlive的Socket连上服务器, 只要两台电脑别断网, 路由器也别断电,
DHCP正常续租, 就这么放着, 过几个小时再用其中一台电脑通过之前建立的TCP连接给另一台发消息, 另一台肯定能收到。

那为什么要有心跳包呢? 其实主要是为了防止上面提到的NAT超时,
既然一些NAT设备判断是否淘汰NAT映射的依据是一定时间没有数据, 那么客户端就主动发一个数据。

心跳包必须由客户端发送, 客户端发现连接断了, 还可以尝试重连服务器。

所以心跳包的主要作用是防止NAT超时, 其次是探测连接是否断开。

链路断开, 没有写操作的TCP连接是感知不到的, 除非这个时候发送数据给服务器, 造成写超时,
否则TCP连接不会知道断开了。主动kill掉一方的进程, 另一方会关闭TCP连接, 是系统代进程给服务器发的FIN。
TCP连接就是这样, 只有明确的收到对方发来的关闭连接的消息, 或者自己意识到发生了写超时, 否则它认为连接还存在。

### 心跳包的时间间隔

每次发送心跳都会消耗电量和网络流量，那么控制心跳包的时间间隔就显得非常重要了。

现实是残酷的, 根据网上的一些说法, 中移动2/3G下, NAT超时时间为5分钟, 中国电信3G则大于28分钟,
理想的情况下, 客户端应当以略小于NAT超时时间的间隔来发送心跳包。

关于如何让心跳间隔逼近NAT超时的间隔, 同时自动适应NAT超时间隔的变化,
可以参看[《移动端IM实践：实现Android版微信的智能心跳机制》](http://www.52im.net/thread-120-1-1.html)

### 服务器如何处理心跳包

如果客户端心跳间隔是固定的, 那么服务器在连接闲置超过这个时间还没收到心跳时, 可以认为对方掉线, 关闭连接。
如果客户端心跳会动态改变, 如上节提到的微信心跳方案, 应当设置一个最大值, 超过这个最大值才认为对方掉线。

还有一种情况就是服务器通过TCP连接主动给客户端发消息出现写超时, 可以直接认为对方掉线。

## 服务器框架

服务器端实现了websocket协议的框架有很多，比如Netty、Undertow、Jetty、Socket.IO等等，
通过性能测试发现Netty表现最好，内存占用非常的少，CPU使用率也不高，尤其内存占用，远远小于其它框架。

## 客户端框架

浏览器端可选择很多，最著名的的就是[Socket.IO](https://socket.io/)

Android端的也很多，包括[Socket.IO](https://socket.io/)、[okhttp](https://github.com/square/okhttp)等等。

有个教程教你如何在浏览器、IOS和Android本地APP中使用websocket：
[using-websockets-in-native-ios-and-android-apps](https://www.varvet.com/blog/using-websockets-in-native-ios-and-android-apps/)

github源代码地址：<https://github.com/elabs/mobile-websocket-example>

更多使用方法参考官网教程，这里就不多讲了。

## 参考文章

* [using-websockets-in-native-ios-and-android-apps](https://www.varvet.com/blog/using-websockets-in-native-ios-and-android-apps/)
* [Android端消息推送总结：实现原理和心跳保活](http://www.52im.net/thread-341-1-1.html)
* [WebSocket 是什么原理？为什么可以实现持久连接？](https://www.zhihu.com/question/20215561)

