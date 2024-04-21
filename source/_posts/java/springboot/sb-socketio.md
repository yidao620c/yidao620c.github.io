---
title: SpringBoot系列 - 集成SocketIO实时通信
date: 2017-07-16 19:09:58 +0800
toc: true
categories: [ java ]
tags: [ spring ]
---

上一篇讲解了基于STOMP协议实现的WebSocket方案，本篇我讲一下Socket.IO的实现方案。

Socket.IO 主要使用WebSocket协议。但是如果需要的话，Socket.io可以回退到几种其它方法，
例如Adobe Flash Sockets、JSONP拉取、或是传统的AJAX拉取，并且提供完全相同的接口。
尽管它可以被用作WebSocket的包装库，它还是提供了许多其它功能，比如广播至多个套接字，存储与不同客户有关的数据，和异步IO操作。

更多请参考 Socket.IO 官网：<https://socket.io/>
<!-- more -->

## 概述

基于 socket.io 来说，采用 node 实现更加合适，本文使用两个后端的Java开源框架实现。

* 服务端使用 [netty-socketio](https://github.com/mrniko/netty-socketio)
* 客户端使用 [socket.io-client-java](https://github.com/socketio/socket.io-client-java)

业务需求是将之前通过轮询方式调动RESTFul API改成使用WebSocket长连接方式，实现要服务器实时的推送消息，另外还要实时监控POS机的在线状态等。

## 引入依赖

```xml
<!-- netty-socketio-->
<dependency>
    <groupId>com.corundumstudio.socketio</groupId>
    <artifactId>netty-socketio</artifactId>
    <version>1.7.13</version>
</dependency>
<dependency>
<groupId>io.netty</groupId>
<artifactId>netty-resolver</artifactId>
<version>4.1.15.Final</version>
</dependency>
<dependency>
<groupId>io.netty</groupId>
<artifactId>netty-transport</artifactId>
<version>4.1.15.Final</version>
</dependency>
<dependency>
<groupId>io.socket</groupId>
<artifactId>socket.io-client</artifactId>
<version>1.0.0</version>
</dependency>
```

## 服务器代码

先来服务端程序爽一把，话不多说，先上代码：

```java
public class NamespaceSocketServer {
    private static final Logger logger = LoggerFactory.getLogger(NamespaceSocketServer.class);

    public static void main(String[] args) {
        /*
         * 创建Socket，并设置监听端口
         */
        Configuration config = new Configuration();
        //设置主机名
        config.setHostname("localhost");
        //设置监听端口
        config.setPort(9092);
        // 协议升级超时时间（毫秒），默认10秒。HTTP握手升级为ws协议超时时间
        config.setUpgradeTimeout(10000);
        // Ping消息超时时间（毫秒），默认60秒，这个时间间隔内没有接收到心跳消息就会发送超时事件
        config.setPingTimeout(180000);
        // Ping消息间隔（毫秒），默认25秒。客户端向服务器发送一条心跳消息间隔
        config.setPingInterval(60000);
        // 连接认证，这里使用token更合适
        config.setAuthorizationListener(new AuthorizationListener() {
            @Override
            public boolean isAuthorized(HandshakeData data) {
                // String token = data.getSingleUrlParam("token");
                // String username = JWTUtil.getSocketUsername(token);
                // return JWTUtil.verifySocket(token, "secret");
                return true;
            }
        });

        final SocketIOServer server = new SocketIOServer(config);

        /*
         * 添加连接监听事件，监听是否与客户端连接到服务器
         */
        server.addConnectListener(new ConnectListener() {
            @Override
            public void onConnect(SocketIOClient client) {
                // 判断是否有客户端连接
                if (client != null) {
                    logger.info("连接成功。clientId=" + client.getSessionId().toString());
                    client.joinRoom(client.getHandshakeData().getSingleUrlParam("appid"));
                } else {
                    logger.error("并没有人连接上。。。");
                }
            }
        });

        /*
         * 添加监听事件，监听客户端的事件
         * 1.第一个参数eventName需要与客户端的事件要一致
         * 2.第二个参数eventClase是传输的数据类型
         * 3.第三个参数listener是用于接收客户端传的数据，数据类型需要与eventClass一致
         */
        server.addEventListener("login", LoginRequest.class, new DataListener<LoginRequest>() {
            @Override
            public void onData(SocketIOClient client, LoginRequest data, AckRequest ackRequest) {
                logger.info("接收到客户端login消息：code = " + data.getCode() + ",body = " + data.getBody());
                // check is ack requested by client, but it's not required check
                if (ackRequest.isAckRequested()) {
                    // send ack response with data to client
                    ackRequest.sendAckData("已成功收到客户端登录请求", "yeah");
                }
                // 向客户端发送消息
                List<String> list = new ArrayList<>();
                list.add("登录成功，sessionId=" + client.getSessionId());
                // 第一个参数必须与eventName一致，第二个参数data必须与eventClass一致
                String room = client.getHandshakeData().getSingleUrlParam("appid");
                server.getRoomOperations(room).sendEvent("login", list.toString());
            }
        });
        //启动服务
        server.start();
    }
}
```

## Android客户端

老规矩，先上代码爽爽

```java
public class SocketClient {
    private static Socket socket;
    private static final Logger logger = LoggerFactory.getLogger(SocketClient.class);

    public static void main(String[] args) throws URISyntaxException {
        IO.Options options = new IO.Options();
        options.transports = new String[]{"websocket"};
        options.reconnectionAttempts = 2;     // 重连尝试次数
        options.reconnectionDelay = 1000;     // 失败重连的时间间隔(ms)
        options.timeout = 20000;              // 连接超时时间(ms)
        options.forceNew = true;
        options.query = "username=test1&password=test1&appid=com.xncoding.apay2";
        socket = IO.socket("http://localhost:9092/", options);
        socket.on(Socket.EVENT_CONNECT, new Emitter.Listener() {
            @Override
            public void call(Object... args) {
                // 客户端一旦连接成功，开始发起登录请求
                LoginRequest message = new LoginRequest(12, "这是客户端消息体");
                socket.emit("login", JsonConverter.objectToJSONObject(message), (Ack) args1 -> {
                    logger.info("回执消息=" + Arrays.stream(args1).map(Object::toString).collect(Collectors.joining(",")));
                });
            }
        }).on("login", new Emitter.Listener() {
            @Override
            public void call(Object... args) {
                logger.info("接受到服务器房间广播的登录消息：" + Arrays.toString(args));
            }
        }).on(Socket.EVENT_CONNECT_ERROR, new Emitter.Listener() {
            @Override
            public void call(Object... args) {
                logger.info("Socket.EVENT_CONNECT_ERROR");
                socket.disconnect();
            }
        }).on(Socket.EVENT_CONNECT_TIMEOUT, new Emitter.Listener() {
            @Override
            public void call(Object... args) {
                logger.info("Socket.EVENT_CONNECT_TIMEOUT");
                socket.disconnect();
            }
        }).on(Socket.EVENT_PING, new Emitter.Listener() {
            @Override
            public void call(Object... args) {
                logger.info("Socket.EVENT_PING");
            }
        }).on(Socket.EVENT_PONG, new Emitter.Listener() {
            @Override
            public void call(Object... args) {
                logger.info("Socket.EVENT_PONG");
            }
        }).on(Socket.EVENT_MESSAGE, new Emitter.Listener() {
            @Override
            public void call(Object... args) {
                logger.info("-----------接受到消息啦--------" + Arrays.toString(args));
            }
        }).on(Socket.EVENT_DISCONNECT, new Emitter.Listener() {
            @Override
            public void call(Object... args) {
                logger.info("客户端断开连接啦。。。");
                socket.disconnect();
            }
        });
        socket.connect();
    }
}
```

## 关于心跳机制

根据 [Socket.IO文档](https://github.com/socketio/engine.io#methods-1) 解释，
客户端会定期发送心跳包，并触发一个ping事件和一个pong事件，如下：

- `ping` Fired when a ping packet is written out to the server.
- `pong` Fired when a pong is received from the server.
  Parameters:
    - `Number` number of ms elapsed since `ping` packet (i.e.: latency)

这里最重要的两个服务器参数如下：

1. pingTimeout (Number): how many ms without a pong packet to consider the connection closed (60000)
1. pingInterval (Number): how many ms before sending a new ping packet (25000).

也就是说握手协议的时候，客户端从服务器拿到这两个参数，一个是ping消息的发送间隔时间，一个是从服务器返回pong消息的超时时间，
客户端会在超时后断开连接。心跳包发送方向是客户端向服务器端发送，以维持在线状态。

## 关于断线和超时

关闭浏览器、直接关闭客户端程序、kill进程、主动执行disconnect方法都会导致立刻产生断线事件。
而客户端把网络断开，服务器端在 `pingTimeout` ms后产生断线事件、客户端在 `pingTimeout` ms后也产生断线事件。

实际上，超时后会产生一个断线事件，叫"disconnect"。客户端和服务器端都可以对这个事件作出应答，释放连接。

客户端代码：

```java
.on(Socket.EVENT_DISCONNECT, new Emitter.Listener() {
    @Override
    public void call(Object... args) {
        logger.info("客户端断开连接啦。。。");
        socket.disconnect();
    }
});
```

连上服务器后，断开网络。超过了心跳超时时间后，产生断线事件。如果客户端不主动断开连接的话，会自动重连，
这时候发现连接不上，又产生连接错误事件，然后重试2次，都失败后自动断开连接了。

下面是客户端日志：

```
SocketClient - 回执消息=服务器已成功收到客户端登录请求,yeah
SocketClient - Socket.EVENT_PING
SocketClient - Socket.EVENT_PONG
SocketClient - 客户端断开连接啦。。。
SocketClient - Socket.EVENT_CONNECT_ERROR
```

服务器端代码：

```java
server.addDisconnectListener(new DisconnectListener() {
    @Override
    public void onDisconnect(SocketIOClient client) {
        System.out.println("服务器收到断线通知... sessionId=" + client.getSessionId());
    }
});
```

服务器逻辑是，如果在心跳超时后，就直接断开这个连接，并且产生一个断开连接事件。

服务器通过netty处理心跳包ping/pong的日志如下：

```
WebSocket08FrameDecoder - Decoding WebSocket Frame opCode=1
WebSocket08FrameDecoder - Decoding WebSocket Frame length=1
WebSocket08FrameEncoder - Encoding WebSocket Frame opCode=1 length=1
```

## 浏览器客户端演示

对于`netty-socketio`有一个demo工程，里面通过一个网页的聊天小程序演示了各种使用方法。

demo地址：[netty-socketio-demo](https://github.com/mrniko/netty-socketio-demo)

## SpringBoot集成

最后重点讲一下如何在SpringBoot中集成。

### 修改配置

首先maven依赖之前已经讲过了，先修改下`application.yml`配置文件来配置下几个参数，比如主机、端口、心跳时间等等。

```yaml
###################  自定义项目配置 ###################
xncoding:
  socket-hostname: localhost
  socket-port: 9096
```

### 添加Bean配置

然后增加一个SocketServer的Bean配置：

```java
@Configuration
public class NettySocketConfig {

    @Resource
    private MyProperties myProperties;

    @Resource
    private ApiService apiService;
    @Resource
    private ManagerInfoService managerInfoService;

    private static final Logger logger = LoggerFactory.getLogger(NettySocketConfig.class);

    @Bean
    public SocketIOServer socketIOServer() {
        /*
         * 创建Socket，并设置监听端口
         */
        com.corundumstudio.socketio.Configuration config = new com.corundumstudio.socketio.Configuration();
        // 设置主机名，默认是0.0.0.0
        // config.setHostname("localhost");
        // 设置监听端口
        config.setPort(9096);
        // 协议升级超时时间（毫秒），默认10000。HTTP握手升级为ws协议超时时间
        config.setUpgradeTimeout(10000);
        // Ping消息间隔（毫秒），默认25000。客户端向服务器发送一条心跳消息间隔
        config.setPingInterval(60000);
        // Ping消息超时时间（毫秒），默认60000，这个时间间隔内没有接收到心跳消息就会发送超时事件
        config.setPingTimeout(180000);
        // 这个版本0.9.0不能处理好namespace和query参数的问题。所以为了做认证必须使用全局默认命名空间
        config.setAuthorizationListener(new AuthorizationListener() {
            @Override
            public boolean isAuthorized(HandshakeData data) {
                // 可以使用如下代码获取用户密码信息
                String username = data.getSingleUrlParam("username");
                String password = data.getSingleUrlParam("password");
                logger.info("连接参数：username=" + username + ",password=" + password);
                ManagerInfo managerInfo = managerInfoService.findByUsername(username);
                // MD5盐
                String salt = managerInfo.getSalt();
                String encodedPassword = ShiroKit.md5(password, username + salt);
                // 如果认证不通过会返回一个Socket.EVENT_CONNECT_ERROR事件
                return encodedPassword.equals(managerInfo.getPassword());
            }
        });

        final SocketIOServer server = new SocketIOServer(config);

        return server;
    }

    @Bean
    public SpringAnnotationScanner springAnnotationScanner(SocketIOServer socketServer) {
        return new SpringAnnotationScanner(socketServer);
    }
}
```

注意，我在`AuthorizationListener`里面通过调用service做了用户名和密码的认证。通过注解方式可以注入service，
执行相应的连接授权动作。

后面还有个`SpringAnnotationScanner`的定义不能忘记。

### 添加消息结构类

预先定义好客户端和服务器端直接传递的消息类型，使用简单的JavaBean即可，比如

```java
public class ReportParam {
    /**
     * IMEI码
     */
    private String imei;
    /**
     * 位置
     */
    private String location;

    public String getImei() {
        return imei;
    }

    public void setImei(String imei) {
        this.imei = imei;
    }

    public String getLocation() {
        return location;
    }

    public void setLocation(String location) {
        this.location = location;
    }
}
```

### 添加消息处理类

这里才是最核心的接口处理类，所有接口处理逻辑都应该写在这里面，我只举了一个例子，就是POS上传位置接口：

```java
/**
 * 消息事件处理器
 *
 * @author XiongNeng
 * @version 1.0
 * @since 2018/1/19
 */
@Component
public class MessageEventHandler {

    private final SocketIOServer server;
    private final ApiService apiService;

    private static final Logger logger = LoggerFactory.getLogger(MessageEventHandler.class);

    @Autowired
    public MessageEventHandler(SocketIOServer server, ApiService apiService) {
        this.server = server;
        this.apiService = apiService;
    }

    //添加connect事件，当客户端发起连接时调用
    @OnConnect
    public void onConnect(SocketIOClient client) {
        if (client != null) {
            String imei = client.getHandshakeData().getSingleUrlParam("imei");
            String applicationId = client.getHandshakeData().getSingleUrlParam("appid");
            logger.info("连接成功, applicationId=" + applicationId + ", imei=" + imei +
                    ", sessionId=" + client.getSessionId().toString() );
            client.joinRoom(applicationId);
            // 更新POS监控状态为在线
            ReportParam param = new ReportParam();
            param.setImei(imei);
            apiService.updateJustState(param, client.getSessionId().toString(), 1);
        } else {
            logger.error("客户端为空");
        }
    }

    //添加@OnDisconnect事件，客户端断开连接时调用，刷新客户端信息
    @OnDisconnect
    public void onDisconnect(SocketIOClient client) {
        String imei = client.getHandshakeData().getSingleUrlParam("imei");
        logger.info("客户端断开连接, imei=" + imei + ", sessionId=" + client.getSessionId().toString());
        // 更新POS监控状态为离线
        ReportParam param = new ReportParam();
        param.setImei(imei);
        apiService.updateJustState(param, "", 2);
        client.disconnect();
    }

    // 消息接收入口
    @OnEvent(value = Socket.EVENT_MESSAGE)
    public void onEvent(SocketIOClient client, AckRequest ackRequest, Object data) {
        logger.info("接收到客户端消息");
        if (ackRequest.isAckRequested()) {
            // send ack response with data to client
            ackRequest.sendAckData("服务器回答Socket.EVENT_MESSAGE", "好的");
        }
    }

    // 广播消息接收入口
    @OnEvent(value = "broadcast")
    public void onBroadcast(SocketIOClient client, AckRequest ackRequest, Object data) {
        logger.info("接收到广播消息");
        // 房间广播消息
        String room = client.getHandshakeData().getSingleUrlParam("appid");
        server.getRoomOperations(room).sendEvent("broadcast", "广播啦啦啦啦");
    }

    /**
     * 报告地址接口
     * @param client 客户端
     * @param ackRequest 回执消息
     * @param param 报告地址参数
     */
    @OnEvent(value = "doReport")
    public void onDoReport(SocketIOClient client, AckRequest ackRequest, ReportParam param) {
        logger.info("报告地址接口 start....");
        BaseResponse result = postReport(param);
        ackRequest.sendAckData(result);
    }

    /*----------------------------------------下面是私有方法-------------------------------------*/
    private BaseResponse postReport(ReportParam param) {
        BaseResponse result = new BaseResponse();
        int r = apiService.report(param);
        if (r > 0) {
            result.setSuccess(true);
            result.setMsg("报告地址成功");
        } else {
            result.setSuccess(false);
            result.setMsg("该POS机还没有入网，报告地址失败。");
        }
        return result;
    }
}
```

## 添加ServerRunner

还有一个步骤就是添加启动器，在SpringBoot启动之后立马执行：

```java
/**
 * SpringBoot启动之后执行
 *
 * @author XiongNeng
 * @version 1.0
 * @since 2017/7/15
 */
@Component
@Order(1)
public class ServerRunner implements CommandLineRunner {
    private final SocketIOServer server;
    private static final Logger logger = LoggerFactory.getLogger(ServerRunner.class);

    @Autowired
    public ServerRunner(SocketIOServer server) {
        this.server = server;
    }

    @Override
    public void run(String... args) throws Exception {
        logger.info("ServerRunner 开始启动啦...");
        server.start();
    }
}
```

## nginx反向代理

要实现通过域名并走标准80或443端口的话，最好使用nginx做反向代理，跟正常的http反向代理基本一致，
不过websocket需要增加一个upgrade的配置。

下面我以一个实际使用例子来说明如何配置nginx的https访问websocket，并且开启301自动http跳转https。

首先要有一个域名，比如`test.enzhico.net`，然后申请letsencrypt的免费证书，这个过程我不讲了，另外的博客文章里面有。

配置如下:

```
map $http_upgrade $connection_upgrade {
    default upgrade;
    '' close;
}

server {
      server_name test.enzhico.net;
      location / {
          proxy_pass http://localhost:9096;
          proxy_read_timeout 300s;
          proxy_set_header Host $host;
          proxy_set_header X-Real-IP $remote_addr;
          proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
          proxy_http_version 1.1;
          proxy_set_header Upgrade $http_upgrade;
          proxy_set_header Connection $connection_upgrade;
      }
      #root /opt/www/test.enzhico.net;
      #index index.html index.htm;
      error_page 404 /404.html;
          location = /40x.html {
      }

      error_page 500 502 503 504 /50x.html;
          location = /50x.html {
      }

    listen 443 ssl; # managed by Certbot
    ssl_certificate /etc/letsencrypt/live/test.enzhico.net/fullchain.pem; # managed by Certbot
    ssl_certificate_key /etc/letsencrypt/live/test.enzhico.net/privkey.pem; # managed by Certbot
    include /etc/letsencrypt/options-ssl-nginx.conf; # managed by Certbot
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem; # managed by Certbot

}

server {
    listen 80;
    server_name test.enzhico.net;
    return 301 https://$host$request_uri; # managed by Certbot
}
```

注意这其中和普通HTTP代理的关键不同是：

```
proxy_set_header Upgrade $http_upgrade;
proxy_set_header Connection $connection_upgrade;
```

## 参考文章

* [Android端与Java服务端交互——SocketIO](http://blog.csdn.net/u011160994/article/details/47153365)
* [Spring Boot 集成 socket.io 后端实现消息实时通信](http://alexpdh.com/2017/09/03/springboot-socketio/)
* [Spring Boot实战之netty-socketio实现简单聊天室](http://blog.csdn.net/sun_t89/article/details/52060946)

## GitHub源码

[springboot-socketio](https://github.com/yidao620c/SpringBootBucket/tree/master/springboot-socketio)

