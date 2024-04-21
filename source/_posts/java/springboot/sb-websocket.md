---
title: SpringBoot系列 - 集成WebSocket实时通信
date: 2017-07-15 12:28:16 +0800
toc: true
categories: [ java ]
tags: [ spring ]
---

WebSocket是 HTML5 开始提供的一种浏览器与服务器间进行全双工通讯的网络技术。
WebSocket 通信协议于2011年被IETF定为标准RFC 6455，WebSocketAPI 被W3C定为标准。
在WebSocket API 中，浏览器和服务器只需要要做一个握手的动作，然后，浏览器和服务器之间就形成了一条快速通道。
两者之间就直接可以数据互相传送。
<!-- more -->

注意特点：

* 为浏览器和服务端提供了双工异步通信的功能，即服务器可以主动向客户端推送信息，客户端也可以主动向服务器发送信息，是真正的双向平等对话，属于服务器推送技术的一种。
* 建立在 TCP 协议之上，服务器端的实现比较容易。
* 与 HTTP 协议有着良好的兼容性。默认端口也是80和443，并且握手阶段采用 HTTP 协议，因此握手时不容易屏蔽，能通过各种 HTTP
  代理服务器。
* 数据格式比较轻量，性能开销小，通信高效。
* 可以发送文本，也可以发送二进制数据。
* 没有同源限制，客户端可以与任意服务器通信。
* 协议标识符是ws（如果加密，则为wss），服务器网址就是 URL。

有多种方式实现WebSocket协议，比如SpringBoot官方推荐的基于STOMP实现，实现起来非常简单，还有通过Socket.IO来实现。
这里我讲一下基于STOMP的实现方案。

## 概述

STOMP：即`Simple Text Orientated Messaging Protocol`，它是一个简单的文本消息传输协议，属于 WebSocket 的子协议，
提供了一个可互操作的连接格式，允许STOMP客户端与任意STOMP消息代理（Broker）进行交互。STOMP协议由于设计简单，
易于开发客户端，因此在多种语言和多种平台上得到广泛地应用。

## 引入依赖

```xml

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-websocket</artifactId>
    <exclusions>
        <exclusion>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-tomcat</artifactId>
        </exclusion>
    </exclusions>
</dependency>
<dependency>
<groupId>org.springframework.boot</groupId>
<artifactId>spring-boot-starter-jetty</artifactId>
</dependency>
```

个人喜好用jetty取代tomcat，所以才这样写。你可以只需要引入`spring-boot-starter-websocket`即可。

## 配置类

```java
@Configuration
@EnableWebSocketMessageBroker
public class WebSocketConfig extends AbstractWebSocketMessageBrokerConfigurer {

    @Override
    public void registerStompEndpoints(StompEndpointRegistry stompEndpointRegistry) {
        stompEndpointRegistry.addEndpoint("/simple")
                .setAllowedOrigins("*") //解决跨域问题
                .withSockJS();
    }

    @Override
    public void configureMessageBroker(MessageBrokerRegistry registry) {
        registry.enableSimpleBroker("/topic");
    }
}
```

关于这个类说明如下：

1. @EnableWebSocketMessageBroker注解表示开启使用STOMP协议来传输基于代理的消息，Broker就是代理的意思。
2. registerStompEndpoints方法表示注册STOMP协议的节点，并指定映射的URL
3. addEndpoint("/simple").withSockJS(); 用来注册STOMP协议节点，同时指定使用SockJS
4. configureMessageBroker方法用来配置消息代理，由于我们是实现推送功能，这里的消息代理是/topic

## 请求消息类

```java
public class RequestMessage {
    private String name;

    public String getName() {
        return name;
    }
}
```

## 响应消息类

```java
public class ResponseMessage {
    private String responseMessage;

    public ResponseMessage(String responseMessage) {
        this.responseMessage = responseMessage;
    }

    public String getResponseMessage() {
        return responseMessage;
    }
}
```

## SpringMVC控制器

```java
@Controller
public class WsController {

    private final SimpMessagingTemplate messagingTemplate;

    @Autowired
    public WsController(SimpMessagingTemplate messagingTemplate) {
        this.messagingTemplate = messagingTemplate;
    }

    @MessageMapping("/welcome")
    @SendTo("/topic/say")
    public ResponseMessage say(RequestMessage message) {
        System.out.println(message.getName());
        return new ResponseMessage("welcome," + message.getName() + " !");
    }

    /**
     * 定时推送消息
     */
    @Scheduled(fixedRate = 1000)
    public void callback() {
        // 发现消息
        DateFormat df = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
        messagingTemplate.convertAndSend("/topic/callback", "定时推送消息时间: " + df.format(new Date()));
    }
}
```

say方法上添加的@MessageMapping注解和我们之前使用的@RequestMapping类似。@SendTo注解表示当服务器有消息需要推送的时候，
会对订阅了@SendTo中路径的浏览器发送消息。

除此之外，我还定义了一个定时推送消息方法，这个方法每隔1秒会主动给订阅了主题`/topic/callback`的客户端推送消息。

到此为止服务器端就编写完成，可以看到服务器的编写非常简单。

## 客户端

这里我使用浏览器的js来测试，先添加三个js文件：

1. STOMP协议的客户端脚本stomp.js
2. SockJS的客户端脚本sock.js
3. 页面dom操作脚本jQuery

可在我的github项目源码里面找到这三个js文件。

下面是演示的页面：

```html

<html>
<head>
    <meta charset="UTF-8"/>
    <title>广播式WebSocket</title>
    <script src="js/sockjs.min.js"></script>
    <script src="js/stomp.js"></script>
    <script src="js/jquery-3.1.1.js"></script>
</head>
<body onload="disconnect()">
<noscript><h2 style="color: #e80b0a;">Sorry，浏览器不支持WebSocket</h2></noscript>
<div>
    <div>
        <button id="connect" onclick="connect();">连接</button>
        <button id="disconnect" disabled="disabled" onclick="disconnect();">断开连接</button>
    </div>

    <div id="conversationDiv">
        <label>输入你的名字</label><input type="text" id="name"/>
        <button id="sendName" onclick="sendName();">发送</button>
        <p id="response"></p>
        <p id="callback"></p>
    </div>
</div>
<script type="text/javascript">
    var stompClient = null;

    function setConnected(connected) {
        document.getElementById("connect").disabled = connected;
        document.getElementById("disconnect").disabled = !connected;
        document.getElementById("conversationDiv").style.visibility = connected ? 'visible' : 'hidden';
        $("#response").html();
        $("#callback").html();
    }

    function connect() {
        var socket = new SockJS('http://localhost:8092/simple');
        stompClient = Stomp.over(socket);
        stompClient.connect({}, function (frame) {
            setConnected(true);
            console.log('Connected:' + frame);
            stompClient.subscribe('/topic/say', function (response) {
                showResponse(JSON.parse(response.body).responseMessage);
            });
            // 另外再注册一下定时任务接受
            stompClient.subscribe('/topic/callback', function (response) {
                showCallback(response.body);
            });
        });
    }

    function disconnect() {
        if (stompClient != null) {
            stompClient.disconnect();
        }
        setConnected(false);
        console.log('Disconnected');
    }

    function sendName() {
        var name = $('#name').val();
        console.log('name:' + name);
        stompClient.send("/welcome", {}, JSON.stringify({'name': name}));
    }

    function showResponse(message) {
        $("#response").html(message);
    }

    function showCallback(message) {
        $("#callback").html(message);
    }
</script>
</body>
</html>
```

页面上面点击"连接"按钮后，开始连接到`/simple`节点。输入名字后点击发送，将向`/welcome`的url发送消息。

同时订阅了两个主题：`/topic/say` 和 `/topic/callback`，会接收到服务器的say方法的返回，以及定时推送消息。

演示效果：

![](https://xnstatic-1253397658.file.myqcloud.com/sb-websocket20.png)

每隔一秒页面都会刷新显示当前时间。同时控制台也会打印定时推送过来的消息。

## Android客户端

对于想要使用Android客户端进行WebSocket通信的，
可参考 [StompProtocolAndroid](https://github.com/NaikSoftware/StompProtocolAndroid)

## GitHub源码

[springboot-websocket](https://github.com/yidao620c/SpringBootBucket/tree/master/springboot-websocket)

