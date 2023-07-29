---
title: WireShark基本使用
date: 2017-01-05 16:33:19 +0800
toc: true
categories: [ 开发工具 ]
tags: [ 网络工具 ]
---

wireshark是非常流行的网络封包分析软件，功能十分强大。可以截取各种网络封包，显示网络封包的详细信息。
使用wireshark的人必须了解网络协议，否则就看不懂wireshark了。
为了安全考虑，wireshark只能查看封包，而不能修改封包的内容，或者发送封包。

wireshark能获取HTTP，也能获取HTTPS，但是不能解密HTTPS，所以wireshark看不懂HTTPS中的内容。
如果是处理HTTP,HTTPS 还是用[Fiddler](http://www.telerik.com/fiddler),
其他协议比如TCP,UDP就用wireshark。
<!-- more -->

## 界面

直接去<https://www.wireshark.org/download.html>下载相应平台版本然后安装即可。

wireshark是捕获机器上的某一块网卡的网络包，当你的机器上有多块网卡的时候，你需要选择一个网卡。
点击Caputre->Options.. 选择正确的网卡，配置Capture Filter，然后点击"Start"按钮

![](https://xnstatic-1253397658.file.myqcloud.com/wireshark01.png)

开始抓包，最新的2.2.5抓包界面如下:

![](https://xnstatic-1253397658.file.myqcloud.com/wireshark02.png)

WireShark 主要分为这几个界面

1. Display Filter(显示过滤器)，用于过滤
2. Packet List Pane(封包列表)，显示捕获到的封包，有源地址和目标地址，端口号。颜色不同
3. Packet Details Pane(封包详细信息), 显示封包中的字段
4. Dissector Pane(16进制数据)
5. Miscellanous(地址栏，杂项)

## 封包列表

No列编号，Time列是开启监控后到接受包的时间，Source是源IP，Destination是目标IP，
Protocal是协议，Length是包长度，Info是包信息。

## 封包详细信息

这个面板是我们最重要的，用来查看协议中的每一个字段。各行信息分别为:

* Frame: 物理层的数据帧概况
* Ethernet II: 数据链路层以太网帧头部信息
* Internet Protocol Version 4: 互联网层IP包头部信息
* Transmission Control Protocol: 传输层的数据段头部信息，此处是TCP
* Hypertext Transfer Protocol: 应用层的信息，此处是HTTP协议

## 过滤器

WireShark使用过程中最方便的就是它的过滤器了。分为两种:

- 捕捉过滤器：用于决定将什么样的信息记录在捕捉结果中，需要在开始捕捉前设置。
- 显示过滤器：在捕捉结果中进行详细查找，他们可以在得到捕捉结果后随意修改。

### 捕捉过滤器（Capture Filter）

参考：https://wiki.wireshark.org/CaptureFilters

 Protocol | Direction | Host(s)       | Value | Logical Operations | Other expression      
----------|-----------|---------------|-------|--------------------|-----------------------
 tcp      | dst       | 10.10.202.254 | 80    | and                | tcp dst 10.2.2.2 3128 

> ***Protocol（协议）***<br/>
> 可能的值: ether, fddi, ip, arp, rarp, decnet, lat, sca, moprc, mopdl, tcp and udp
> 如果没有特别指明是什么协议，则默认使用所有支持的协议。<br/>
> ***Direction（方向）***<br/>
> 可能的值: src, dst, src and dst, src or dst
> 如果没有特别指明来源或目的地，则默认使用 "src or dst" 作为关键字。
> 例如，"host 10.2.2.2"与"src or dst host 10.2.2.2"是一样的。<br/>
> ***Host(s)***<br/>
> 可能的值： net, port, host, portrange
> 如果没有指定此值，则默认使用"host"关键字。
> 例如，"src 10.1.1.1"与"src host 10.1.1.1"相同。<br/>
> ***Logical Operations（逻辑运算）***<br/>
> 可能的值：not, and, or
> 否("not")具有最高的优先级。或("or")和与("and")具有相同的优先级，运算时从左至右进行。

例如:

```
"not tcp port 3128 and tcp port 23"与"(not tcp port 3128) and tcp port 23"相同
"not tcp port 3128 and tcp port 23"与"not (tcp port 3128 and tcp port 23)"不同
```

### 显示过滤器（Display Filter）

参考：https://wiki.wireshark.org/DisplayFilters

显示过滤器就是在用来过滤封包列表的，它不影响捕获的日志文件，只是用来过滤显示列表中显示的项目用，
是在运行时设置的随时可以更改。

示例：

```
ip.src != 10.1.2.3 and ip.dst != 10.4.5.6
ip.dst == 14.215.177.38 and ip.src == 10.10.111.230 and ssl
```

## TCP三次握手

这个在TCP/IP协议里面讲的太多了，就不多说了，直接给个图就能看得懂:

![](https://xnstatic-1253397658.file.myqcloud.com/tcp01.png)

那么通过抓取一次HTTP访问例子来说明这个握手过程，
比如要访问<http://www.cnblogs.com>，先通过ping命令拿到ip地址为"223.6.248.220"，
然后新建一个Capture Filter，规则为`host 223.6.248.220`，运行后网页访问这个链接:

![](https://xnstatic-1253397658.file.myqcloud.com/tcp02.png)

非常清楚的显示了三次TCP握手过程。

## 跟踪一个请求

在wireshark中输入http过滤，然后选中GET / HTTP/1.1的那条记录，右键然后点击"Follow TCP Stream",
这样做的目的是为了得到与浏览器打开网站相关的数据包

![](https://xnstatic-1253397658.file.myqcloud.com/wireshark03.png)

如果要跟踪HTTPS请求，先在Edit->Preferences->Protocols里面选择SSL

## Wireshark解密TLS1.2数据包

对于HTTPS通信的数据包，如果想要Wireshark在抓包过程中自动解密，则要求密钥套件算法中不能使用DH或DHE算法。 可将Nginx的密钥套件配置修改成如下：

```nginx
ssl_ciphers HIGH:!aNULL:!DH;!DHE:!ECDH;
```

下载Nginx的私钥或者导出私钥和证书为nginx.p12格式的正式库文件。 然后下载最新的Wireshark3.2.7，选择`编辑 》 首选项 》 protocols 》 TLS`，配置私钥或PCKS12格式的证书库即可。
