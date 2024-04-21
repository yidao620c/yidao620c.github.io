---
title: Jetty9简易教程
date: 2017-06-12 20:12:08 +0800
toc: true
categories: [ web ]
tags: [ web ]
---

Jetty是一个用 Java 实现、开源、基于标准的，并且具有丰富功能的 Http 服务器和 Web 容器。
Jetty 可以用来作为一个传统的 Web 服务器，也可以作为一个动态的内容服务器，并且 Jetty 可以非常容易的嵌入到 Java 应用程序当中。

jetty是轻量级的web服务器和servlet引擎。它的最大特点是：可以很方便的作为嵌入式服务器。
就是只要引入jetty的jar包，可以通过直接调用其API的方式来启动web服务。
<!-- more -->

Jetty的广泛应用得益于其诸多优秀的特性：

1. 轻量级：Jetty体积小巧，占用系统资源较少。
2. 易嵌入性：Jetty既可以像tomcat一样独立运行，也可以很方便的嵌入到工具、框架或其他应用服务器中运行。
   Jetty在设计之初就是作为一个可以嵌入到其他的Java代码中的servlet容器而设计的，因此开发小组将Jetty作为一组Jar文件提供出来，
   可以非常方便的在自己的容器中将Jetty实例化成一个对象并操纵该容器对象。
3. 灵活性：Jetty的体系架构及其面向接口的设计实现了功能模块高度可插拔和可扩展的特性，可以非常方便的根据需要来配置Jetty启用的功能。
4. 稳定性：Jetty运行速度较快，即使有大量服务请求并发的情况下，系统性能也能保持在一个可以接受的状态。

本篇简单介绍下如何使用Jetty9以及跟maven、idea等的集成。

## Jetty的体系结构

先介绍Jetty的大致架构：

![](https://xnstatic-1253397658.file.myqcloud.com/jetty01.jpg)

下面分别对上图中的几个部分作简要介绍：

* Connector负责解析服务器请求并产生应答，不同的Connector用于处理不同协议的请求。
* Handler用于处理经过Connector解析的请求并产生应答内容，同样可以通过配置不同的Handler来负责处理不同的请求。
* TheadPool：管理和调度多个线程，用于服务于多个连接请求。
* Server代表一个Jetty服务器对象，主要作用是协同Connector、Handler和TheadPool的工作。

其中，TheadPool可以根据配置选择是否使用，Connector和Handler也可以通过配置非常方便的实现替换。

## 安装

去[jetty 官网](http://www.eclipse.org/jetty/download.html)下载最新的zip压缩包，我下的是9.4.6.v20170531版本。
然后解压缩到某个目录，配置下系统环境变量JETTY_HOME，并将bin目录加入到系统PATH中去即可。

## 启动

jetty的启动跟Tomcat不同，cd到JETTY_HOME目录，执行：

```bash
java -jar start.jar
```

打开浏览器，访问http://127.0.0.1:8080，此时可以看到Jetty的欢迎页面了。

## 目录结构

介绍一下jetty的目录，跟tomcat容器一样，我们也需要了解各个目录是做什么的

 Dir         | Description                                       
-------------|---------------------------------------------------
 bin/        | 存放在Unix系统下运行的shell脚本                              
 ect/        | 配置文件                                              
 lib/        | 运行所依赖的jar文件                                       
 logs/       | 运行时的日志文件                                          
 modules/    | 各个模块                                              
 resources/  | 包含新增到classpath配置文件夹，如log4j.properties             
 webapps/    | 存放Web应用，Jetty会自动加载这个目录下的所有Web应用                   
 start.jar   | 运行Jetty的jar。在命令行环境下以 java -jar start.jar 来启动Jetty 
 start.ini   | 存放启动信息                                            
 notice.html | 许可信息等                                             
 README.txt  | 一些有用的说明                                           
 VERSION.txt | 版本信息                                              

修改端口号现在jetty9是修改`start.ini`文件中的`jetty.port=8001`

## 发布

默认的webapps目录下面没有ROOT目录，最好先创建一个。
自己写的web项目，直接解压到webapps目录下面的ROOT目录，或者另外一个名字叫myapp的目录，通过命令：

```
java -jar start.jar jetty.port=8002
```

来启动项目

对于发布到ROOT目录的访问URL：localhost:8002

对于发布到myapp目录的访问URL：localhost:8002/myapp

## 与IDEA的集成

最好的方式就是使用maven的插件`jetty-maven-plugin`来集成：

```xml
<plugin>
    <groupId>org.eclipse.jetty</groupId>
    <artifactId>jetty-maven-plugin</artifactId>
    <version>${jetty.version}</version>
    <configuration>
        <scanIntervalSeconds>10</scanIntervalSeconds>
        <webApp>
            <contextPath>/</contextPath>
            <webInfIncludeJarPattern>.*/^(asm-all-repackaged)[^/]*\.jar$</webInfIncludeJarPattern>
        </webApp>
        <httpConnector>
            <port>8080</port>
        </httpConnector>
        <!--Close jetty Automatic deployment and spring-loaded conflict -->
        <reload>manual</reload>
    </configuration>
</plugin>
```

这时候展开右端的maven菜单，在Plugins下面有jetty，点击里面的`jetty:run`或`jetty:run-exploded`即可启动开发环境了。

## 嵌入式容器

Jetty还能通过嵌入式的方式使用，有API方式和maven插件方式。

### API方式

添加maven依赖

```xml
<dependency>
  <groupId>org.eclipse.jetty</groupId>
  <artifactId>jetty-webapp</artifactId>
  <version>9.3.2.v20150730</version>
  <scope>test</scope>
</dependency>
<dependency>
  <groupId>org.eclipse.jetty</groupId>
  <artifactId>jetty-annotations</artifactId>
  <version>9.3.2.v20150730</version>
  <scope>test</scope>
</dependency>
<dependency>
  <groupId>org.eclipse.jetty</groupId>
  <artifactId>apache-jsp</artifactId>
  <version>9.3.2.v20150730</version>
  <scope>test</scope>
</dependency>
<dependency>
  <groupId>org.eclipse.jetty</groupId>
  <artifactId>apache-jstl</artifactId>
  <version>9.3.2.v20150730</version>
  <scope>test</scope>
</dependency>
```

官方的启动代码

```java
public class SplitFileServer
{
    public static void main( String[] args ) throws Exception
    {
        // 创建Server对象，并绑定端口
        Server server = new Server();
        ServerConnector connector = new ServerConnector(server);
        connector.setPort(8090);
        server.setConnectors(new Connector[] { connector });

        // 创建上下文句柄，绑定上下文路径。这样启动后的url就会是:http://host:port/context
        ResourceHandler rh0 = new ResourceHandler();
        ContextHandler context0 = new ContextHandler();
        context0.setContextPath("/");

        // 绑定测试资源目录（在本例的配置目录dir0的路径是src/test/resources/dir0）
        File dir0 = MavenTestingUtils.getTestResourceDir("dir0");
        context0.setBaseResource(Resource.newResource(dir0));
        context0.setHandler(rh0);

        // 和上面的例子一样
        ResourceHandler rh1 = new ResourceHandler();
        ContextHandler context1 = new ContextHandler();
        context1.setContextPath("/");
        File dir1 = MavenTestingUtils.getTestResourceDir("dir1");
        context1.setBaseResource(Resource.newResource(dir1));
        context1.setHandler(rh1);

        // 绑定两个资源句柄
        ContextHandlerCollection contexts = new ContextHandlerCollection();
        contexts.setHandlers(new Handler[] { context0, context1 });
        server.setHandler(contexts);

        // 启动
        server.start();

        // 打印dump时的信息
        System.out.println(server.dump());

        // join当前线程
        server.join();
    }
}
```

### maven插件方式

推荐这种方式，实在太方便了，这种方式就是我上面通过`jetty-maven-plugin`和IDEA集成的方式，不再多说。

命令行的maven启动方式：

```
mvn jetty:run
```

