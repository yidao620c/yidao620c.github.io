---
title: SpringBoot系列 - 集成Echarts导出图片
date: 2017-08-19 15:17:28 +0800
toc: true
categories: [ java ]
tags: [ spring ]
---

Echarts是百度一款开源可视化图表库，基于html5 Canvas的。能够快速让你看到漂亮的效果。也是百度开源产品中的良心之作。

有时候在Java程序中也需要导出好看的图表，比如我经常会基于JMH做各种微基准测试，想将测试结果可视化导出为图表形式。
试用了一下JFreeChart，跟Echarts导出的图比起来还是弱了不少。但是Echarts是基于js的，只能在浏览器中解析和导出图片，怎么办呢？

后来我想到一个方法，就是基于WebSocket技术，服务器将图表数据推送到页面，然后页面再触发导出动作。
本篇将介绍如何通过SpringBoot、SocketIO、Echarts技术来实现这个图表导出功能。
<!-- more -->

大致步骤是这样的：

1. 编写一个RESTful API接口，用来接收生成图表需要的数据，然后向页面推送`导出图片`的消息请求。
1. 编写一个WebSocket接口，接收Base64格式的图片编码，然后将其转换为图片保存到磁盘上
1. 启动应用后打开websocket服务端，并监听页面发送的`Echarts图片导出`消息
1. 打开页面，自动连接上websocket服务器，并监听`导出图片`的消息请求，收到消息后发送`Echarts图片导出`消息
1. 后面就可以调用这个RESTful API接口来讲数据可视化为Echarts图片并保存了。

我曾经尝试过，Java程序通过phantomjs启动浏览器打开页面，最后导出来的图片有1.5M，而直接用浏览器打开后导出大小只有50K，
所以放弃了phantomjs方案。

## maven依赖

```xml

<properties>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
    <java.version>1.8</java.version>
    <netty.version>4.1.19.Final</netty.version>
    <thymeleaf.version>3.0.7.RELEASE</thymeleaf.version>
    <thymeleaf-layout-dialect.version>2.2.2</thymeleaf-layout-dialect.version>
    <jmh.version>1.20</jmh.version>
</properties>

<dependencies>

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
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
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-thymeleaf</artifactId>
</dependency>
<dependency>
    <groupId>org.apache.commons</groupId>
    <artifactId>commons-lang3</artifactId>
    <version>3.7</version>
</dependency>
<dependency>
    <groupId>commons-io</groupId>
    <artifactId>commons-io</artifactId>
    <version>2.5</version>
</dependency>
<dependency>
    <groupId>commons-codec</groupId>
    <artifactId>commons-codec</artifactId>
    <version>1.11</version>
</dependency>
<dependency>
    <groupId>junit</groupId>
    <artifactId>junit</artifactId>
    <version>4.12</version>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.hamcrest</groupId>
    <artifactId>hamcrest-all</artifactId>
    <version>1.3</version>
    <scope>test</scope>
</dependency>

<!-- netty-socketio-->
<dependency>
    <groupId>com.corundumstudio.socketio</groupId>
    <artifactId>netty-socketio</artifactId>
    <version>1.7.13</version>
</dependency>
<dependency>
    <groupId>io.netty</groupId>
    <artifactId>netty-buffer</artifactId>
    <version>${netty.version}</version>
</dependency>
<dependency>
    <groupId>io.netty</groupId>
    <artifactId>netty-codec</artifactId>
    <version>${netty.version}</version>
</dependency>
<dependency>
    <groupId>io.netty</groupId>
    <artifactId>netty-codec-http</artifactId>
    <version>${netty.version}</version>
</dependency>
<dependency>
    <groupId>io.netty</groupId>
    <artifactId>netty-common</artifactId>
    <version>${netty.version}</version>
</dependency>
<dependency>
    <groupId>io.netty</groupId>
    <artifactId>netty-handler</artifactId>
    <version>${netty.version}</version>
</dependency>
<dependency>
    <groupId>io.netty</groupId>
    <artifactId>netty-resolver</artifactId>
    <version>${netty.version}</version>
</dependency>
<dependency>
    <groupId>io.netty</groupId>
    <artifactId>netty-transport</artifactId>
    <version>${netty.version}</version>
</dependency>
<dependency>
    <groupId>io.netty</groupId>
    <artifactId>netty-transport-native-epoll</artifactId>
    <version>${netty.version}</version>
</dependency>
<dependency>
    <groupId>io.netty</groupId>
    <artifactId>netty-transport-native-unix-common</artifactId>
    <version>${netty.version}</version>
</dependency>
<dependency>
    <groupId>io.socket</groupId>
    <artifactId>socket.io-client</artifactId>
    <version>1.0.0</version>
</dependency>

</dependencies>
```

## 配置文件

修改application.yml，配置web端口、socket服务端口、图片保存路径等：

```yaml
###################  自定义项目配置 ###################
xncoding:
  socket-port: 9076    #socket端口
  ping-interval: 60000 #Ping消息间隔（毫秒）
  ping-timeout: 180000 #Ping消息超时时间（毫秒）
  image-dir: E:/pics/

###################  项目启动端口  ###################
server:
  port: 9075
  jetty:
    max-http-post-size: 20000000
```

## WebSocket服务

基于Socket.IO来实现WebSocket服务，关于这个我专门写了一篇博客：[集成SocketIO实时通信](https://www.xncoding.com/2017/07/16/spring/sb-socketio.html)

这里我不再多讲怎么集成，我只贴一下监听方法：

```java
/**
 * 保存客户端传来的图片数据
 *
 * @param client     客户端
 * @param ackRequest 回执消息
 * @param imgData    Base64的图形数据
 */
@OnEvent(value = "savePic")
public void onSavePic(SocketIOClient client, AckRequest ackRequest, String imgData) {
    logger.info("保存客户端传来的图片数据 start, sessionId=" + client.getSessionId().toString());
    String r = apiService.saveBase64Pic(imgData);
    logger.info("保存客户端传来的图片 = {}", r);
    ackRequest.sendAckData("图片保存结果=" + r);
}
```

这里WebSocket服务器会监听消息`savePic`，参数为Base64的图形数据，然后将其保存到磁盘上面。

## Echarts配置类

Echarts可以到处非常多类型的图表，每个图表初始化需要一个特定json对象，里面的配置不一样。我使用了其中的一种用来显示性能测试对比的纵向柱状图。
然后需要编写这个配置对象，包括各个内嵌对象，具体我就不贴出来了，参考最后面的github源码。

## 图片导出接口

接下来编写对外开放的图片导出RESTful API接口：

```java
/**
 * 对外API接口类
 */
@RestController
@RequestMapping(value = "/api/v1")
public class PublicController {

    @Resource
    private ApiService apiService;

    private static final Logger _logger = LoggerFactory.getLogger(PublicController.class);
    
    @RequestMapping(value = "/data", method = RequestMethod.POST)
    public ResponseEntity<BaseResponse> doJoin(HttpServletRequest request) throws Exception {
        _logger.info("数据上传消息push接口 start....");
        String jsonBody = IOUtils.toString(request.getInputStream(), Charset.forName("UTF-8"));
        EchartsData echartsData = new EchartsData("", jsonBody);
        String jsonString = new ObjectMapper().writeValueAsString(echartsData);
        apiService.pushMsg("notify", jsonString);
        BaseResponse result = new BaseResponse<>(true, "数据上传消息push成功", null);
        return new ResponseEntity<>(result, HttpStatus.OK);
    }
}
```

上面的pushMsg方法定义：

```java
/**
 * 服务器主动推送消息
 *
 * @param msgType 消息类型
 * @param jsonData echarts图表数据
 */
public void pushMsg(String msgType, String jsonData) {
    SocketIOClient targetClient = this.server.getClient(UUID.fromString(sessionId));
    if (targetClient == null) {
        logger.error("sessionId=" + sessionId + "在server中获取不到client");
    } else {
        targetClient.sendEvent(msgType, jsonData);
    }
}
```

服务器将数据转换成echarts的配置json字符串后，就给浏览器推送一个`notify`的消息。

## 页面客户端

接下来编写js客户端来连接WebSocket服务，并监听`notify`的消息，在页面上生成Echarts图表后，
再给服务器发送一个`savePic`的消息，并把图表的Base64编码数据传过去。

```html
<!DOCTYPE html>
<html xmlns="http://www.w3.org/1999/xhtml"
      xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <meta name="renderer" content="webkit">
    <title>Echarts Demo</title>
    <link rel="shortcut icon" th:href="@{/favicon.ico}"/>
    <link th:href="@{/static/css/bootstrap.css}" rel="stylesheet"/>
    <style>
        body {
            padding: 20px;
        }

        #console {
            height: 100px;
            overflow: auto;
        }

        .connect-msg {
            color: green;
        }

        .disconnect-msg {
            color: red;
        }

        .send-msg {
            color: #888
        }
    </style>
</head>
<body>
<h1>Netty-socketio Demo Chat</h1>
<br/>
<div id="console" class="well"></div>

<!-- 为 ECharts 准备一个具备大小（宽高）的 DOM -->
<div id="main" style="width:860px; height:470px;"></div>

<form class="well form-inline" onsubmit="return false;">
    <input id="msg" class="input-xlarge" type="text" placeholder="Type something..."/>
    <button type="button" onClick="sendDisconnect()" class="btn">Disconnect</button>
    <button type="button" onClick="sendSavePic()" class="btn">Save Picture</button>
</form>

<script th:src="@{/static/js/jquery-1.10.1.min.js}"></script>
<script th:src="@{/static/js/socket.io.js}"></script>
<script th:src="@{/static/js/moment.min.js}"></script>
<script th:src="@{/static/js/echarts.common.min.js}" charset="utf-8"></script>

<script>
    var socket;
    var myChart;

    function sendDisconnect() {
        socket.disconnect();
    }

    function output(message) {
        var currentTime = "<span class='time'>" + moment().format('HH:mm:ss.SSS') + "</span>";
        var element = $("<div>" + currentTime + " " + message + "</div>");
        $('#console').prepend(element);
    }

    function sendSavePic() {
        socket.emit('savePic', myChart.getDataURL(), function (result) {
            output('<span class="connect-msg">' + result + '</span>');
        });
    }

    function initPage() {
        console.log('this is index.html log...');

        var userName = 'user' + Math.floor((Math.random() * 1000) + 1);
        socket = io.connect('http://localhost:9076');

        console.log('socket = ' + socket);

        socket.on('connect', function () {
            console.log('connect successful');
            output('<span class="connect-msg">Client has connected to the server!</span>');
        });

        socket.on('disconnect', function () {
            console.log('disconnect successful');
            output('<span class="disconnect-msg">The client has disconnected!</span>');
        });

        socket.on('notify', function (jsonBody) {
            console.log('get notify message from server...');
            var msg = JSON.parse(jsonBody);
            output('<span class="connect-msg">接收到notify消息, 去给我保存图片</span>');
            var option = msg.option;
            // 使用刚指定的配置项和数据显示图表。
            myChart.setOption(JSON.parse(option));
            // 渲染完成之后再给服务器发送保存图片消息
            sendSavePic();
        });

        /**************************************分割线***********************************/
        // 基于准备好的dom，初始化echarts实例
        myChart = echarts.init(document.getElementById('main'));
        console.log('index initPage finished!')
    }

    $(function () {
        initPage();
    });

</script>

</body>

</html>
```

## 测试

编写测试类：

````java
public class ApplicationTests {
    @Test
    public void testOption() throws Exception {
        String titleStr = "对象序列化为JSON字符串";
        // 几个测试对象
        List<String> objects = Arrays.asList("FastJson", "Jackson", "Gson", "Json-lib");
        // 测试维度，输入值n
        List<String> dimensions = Arrays.asList("10000次", "100000次", "1000000次");
        // 有几个测试对象，就有几组测试数据，每组测试数据中对应几个维度的结果
        List<List<Double>> allData = new ArrayList<List<Double>>(){
            {
               add(Arrays.asList(2.17, 9.10, 21.70));
               add(Arrays.asList(1.94, 8.94, 19.43));
               add(Arrays.asList(4.88, 22.88, 48.89));
               add(Arrays.asList(9.11, 58.14, 108.44));
            }
        };
        String optionStr = generateOption(titleStr, objects, dimensions, allData, "秒");
        // POST到接口上
        postOption(optionStr, "http://localhost:9075/api/v1/data");
    }
}
````

一切准备就绪后，就可以开始测试了，步骤如下：

1. 启动应用
2. 打开首页 <http://localhost:9075/>
3. 运行测试类`ApplicationTests.java`

结果首页显示如下：

![](https://xnstatic-1253397658.file.myqcloud.com/sb-echarts01.png)

同时去`E:/pics/`目录去看看，图片也成功保存。

## GitHub源码

[springboot-echarts](https://github.com/yidao620c/SpringBootBucket/tree/master/springboot-echarts)

