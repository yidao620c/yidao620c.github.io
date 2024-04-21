---
title: RabbitMQ简易教程 - WebSocket
date: 2017-05-17 08:24:25 +0800
toc: true
categories: [ 中间件 ]
tags: [ rabbitmq ]
---

之前写过一篇 [使用Ajax实现异步任务](https://www.xncoding.com/2017/05/02/web/async.html) 的文章，
介绍了对于需要知道异步处理返回结果的情况，使用Ajax的轮训和长连接方式实现。
但是这两种方式都会生成大量的HTTP连接，对服务器资源是一种巨大的浪费，
这里正式介绍如果通过`WebSocket` + `RabbitMQ`来优雅的实现。
<!-- more -->

## WebSocket 双向通信

WebSocket 是 HTML5 中一种新的通信协议，能够实现浏览器与服务器之间全双工通信。
如果浏览器和服务端都支持 WebSocket 协议的话，该方式实现的消息推送无疑是最高效、简洁的。
并且最新版本的 IE、Firefox、Chrome 等浏览器都已经支持 WebSocket 协议，
Apache Tomcat 7.0.27 以后的版本也开始支持 WebSocket。

## 基于 RabbitMQ 的实时消息推送

RabbitMQ 有很多第三方插件，可以在 AMQP 协议基础上做出许多扩展的应用。
Web STOMP 插件就是基于 AMQP 之上的 STOMP 文本协议插件，
利用 WebSocket 能够轻松实现浏览器和服务器之间的实时消息传递。

首先打开`Web STOMP` 插件：

```bash
rabbitmq-plugins enable rabbitmq_web_stomp
```

这里我通过一个实际的例子来演示怎样通过websocket实现异步任务的页面自动刷新。

大致步骤：

1. 后端通过一个python程序向名字为`rabbitmq`的exchange发送消息，并设置routing key为`zjj`
2. 页面通过js，创建一个自动删除的队列，并且将这个队列跟这个`rabbitmq`的exchange进行绑定，binding key为`zjj`
3. 页面初始显示`loading...`，当收到消息后将内容更新为消息字符串。

RabbitMQ Web STOMP 插件可以理解为 HTML5 WebSocket 与 STOMP 协议间的桥接，
目的也是为了让浏览器能够使用 RabbitMQ。当 RabbitMQ 消息服务器开启了 STOMP 和 Web STOMP 插件后，浏览器端就可以轻松地使用
WebSocket 或者 SockerJS 客户端实现与 RabbitMQ 服务器进行通信。

RabbitMQ Web STOMP 是对 STOMP 协议的桥接，因此其语法也完全遵循 STOMP 协议。
STOMP 是基于 frame 的协议，与 HTTP 的 frame 相似。
一个 Frame 包含一个 command，一系列可选的 headers 和一个 body。
STOMP client 的用户代理可以充当两个角色，当然也可能同时充当：
作为生产者，通过 SEND frame 发送消息到服务器；
作为消费者，发送 SUBCRIBE frame 到目的地并且通过 MESSAGE frame 从服务器获取消息。

在 Web 页面中利用 WebSocket 使用 STOMP 协议只需要下载 `stomp.js` 即可，
考虑到老版本的浏览器不支持 WebSocket，SockJS 则提供了 WebSocket 的模拟支持。

web页面代码：

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Echarts demo</title>
    <link rel="stylesheet" href="../../bootstrap/css/bootstrap.min.css">
    <!-- 引入 ECharts 文件 -->
    <script type="text/javascript" src="../../js/jquery-1.12.4.min.js"></script>
    <script type="text/javascript" src="./sockjs.min.js"></script>
    <script type="text/javascript" src="./stomp.min.js"></script>
</head>
<body>

<h2 style="margin-left: 15px;">测试Websocket自动刷新页面状态</h2>

<div class="row" style="margin-left: 0">
    <div class="col-md-3" style="font-size: large">
        <span>状态更新：</span>
        <span id="load" style="color: #5bc0de">loading...</span>
    </div>
</div>

<script type="text/javascript">
    $(function () {
        console.log('starting...');
        // Stomp.js boilerplate
        var ws;
        if (typeof (WebSocket) === "function") {
            console.log('ws协议...');
            ws = new WebSocket('ws://192.168.217.226:15674/ws');
        } else {
            console.log('http协议...');
            ws = new SockJS('http://192.168.217.226:15674/stomp');
        }
        // Init Client
        var client = Stomp.over(ws);
        // SockJS does not support heart-beat: disable heart-beats
        // client will send heartbeats every xxxms
        client.heartbeat.outgoing = 0;
        // client does not want to receive heartbeats
        client.heartbeat.incoming = 0;
        // Declare on_connect
        var on_connect = function (x) {
            client.subscribe("/exchange/rabbitmq/zjj", function (d) {
                // update result
                $("#load").empty().text(d.body);
                // disconnect client
                // when close the brower, it will be closed automatically too
                client.disconnect(function () {
                    console.log("See you next time!");
                });
            });
        };
        // Declare on_error
        var on_error = function () {
            console.log('error');
        };
        // Conect to RabbitMQ
        client.connect('guest', 'guest', on_connect, on_error, '/');
    });

</script>

</body>
</html>
```

刚开始打开的效果：

![](https://xnstatic-1253397658.file.myqcloud.com/rb09.png)

web页面控制台日志如下，说明已经连上了队列：

![](https://xnstatic-1253397658.file.myqcloud.com/rb10.png)

同时我去看rabbitmq管理界面，发现生成了一个随机队列：

![](https://xnstatic-1253397658.file.myqcloud.com/rb14.png)

后台python发送消息代码：

```python send.py
import pika
import sys

connection = pika.BlockingConnection(pika.ConnectionParameters(host='192.168.217.226', port=5673))
channel = connection.channel()

channel.exchange_declare(exchange='rabbitmq',
                         type='topic')

routing_key = 'zjj'
message = '[disk.info] Hello World!333'
channel.basic_publish(exchange='rabbitmq',
                      routing_key=routing_key,
                      body=message)
print(" [x] Sent %r:%r" % (routing_key, message))

connection.close()
```

先运行后台消息发送：`python send.py`

然后看看页面，变成这样了：

![](https://xnstatic-1253397658.file.myqcloud.com/rb11.png)

web页面控制台情况：

![](https://xnstatic-1253397658.file.myqcloud.com/rb12.png)

在看rabbitmq的管理界面：

![](https://xnstatic-1253397658.file.myqcloud.com/rb13.png)

可以看出，当我把websocket连接断了之后，那个queue自动删了，
另外把页面关闭的时候websocket连接也会自动关闭。

## 关于destination

使用`Web STOMP`的时候，最核心的两个东西是发送和订阅消息：

```js
// 发送消息
client.send(destination, head, body);
// 订阅消息
client.subcribe(destination, callback);
```

这里需要详细讲解`destination`的含义，根据使用场景的不同，主要有以下4种：

1. /exchange/<exchangeName>

对于 SUBCRIBE frame，destination 一般为/exchange/<exchangeName>/[/pattern] 的形式。
该 destination 会创建一个唯一的、自动删除的随机queue，
并根据 pattern 将该 queue 绑定到所给的 exchange，实现对该队列的消息订阅。

对于 SEND frame，destination 一般为/exchange/<exchangeName>/[/routingKey] 的形式。
这种情况下消息就会被发送到定义的 exchange 中，并且指定了 routingKey。

2. /queue/<queueName>

对于 SUBCRIBE frame，destination 会定义<queueName>的共享 queue，并且实现对该队列的消息订阅。

对于 SEND frame，destination 只会在第一次发送消息的时候会定义<queueName>的共享 queue。
该消息会被发送到默认的 exchange 中，routingKey 即为<queueName>。

3. /amq/queue/<queueName>

这种情况下无论是 SUBCRIBE frame 还是 SEND frame 都不会产生 queue。
但如果该 queue 不存在，SUBCRIBE frame 会报错。

对于 SUBCRIBE frame，destination 会实现对队列<queueName>的消息订阅。
对于 SEND frame，消息会通过默认的 exhcange 直接被发送到队列<queueName>中。

4. /topic/<topicName>

对于 SUBCRIBE frame，destination 创建出自动删除的、非持久的 queue 并根据 routingkey 为<topicName>
绑定到 amq.topic exchange 上，同时实现对该 queue 的订阅。

对于 SEND frame，消息会被发送到 amq.topic exchange 中，routingKey 为<topicName>。

对于在页面发送消息的例子跟订阅类似，这里就不再演示。

## 最后建议

WebSocket 作为 HTML5 提供的新一代客户端-服务器异步通信方法，能够轻松完成前端与后台的双向通信。
RabbitMQ 服务提供了一个 STOMP 插件，能够实现与 WebSocket 的桥接，这样既能够实现消息的主动推送，
同时也能够实现消息的异步处理。

在传统的 Web 开发中存在许多状态变更实时性的需求，比如资源被占用后需要广播它的实时状态，
利用本文提出的解决方案，可以方便将其推送到所有监听的客户端。
因此在开发新的 Web 项目中，建议使用本文提出的方案替代原来 ajax 轮询或长连接刷新状态。

