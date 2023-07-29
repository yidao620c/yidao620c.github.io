---
title: Spring集成Axis2
date: 2017-09-06 15:22:10 +0800
toc: true
categories: [ java ]
tags: [ spring ]
---

虽然现在做WebService开发的越来越少了，但是还是会碰到很多老的系统用到这个技术。
本篇讲解一下如何在Spring4里面集成Axis2开发WebService，包括服务器端和客户端。
<!-- more -->

## 服务端

服务端采用Spring4 MVC技术，maven工程

### 添加依赖

```xml

<properties>
    <spring.version>4.3.13.RELEASE</spring.version>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
    <axis2-version>1.7.7</axis2-version>
</properties>

<dependencies>
<!-- spring start-->
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-core</artifactId>
    <version>${spring.version}</version>
</dependency>
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-web</artifactId>
    <version>${spring.version}</version>
</dependency>
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-webmvc</artifactId>
    <version>${spring.version}</version>
</dependency>
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-beans</artifactId>
    <version>${spring.version}</version>
</dependency>
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-context</artifactId>
    <version>${spring.version}</version>
</dependency>
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-context-support</artifactId>
    <version>${spring.version}</version>
</dependency>
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-test</artifactId>
    <version>${spring.version}</version>
    <scope>test</scope>
</dependency>

<!-- axis2 -->
<dependency>
    <groupId>org.apache.axis2</groupId>
    <artifactId>axis2</artifactId>
    <version>${axis2-version}</version>
</dependency>
<dependency>
    <groupId>org.apache.axis2</groupId>
    <artifactId>axis2-spring</artifactId>
    <version>${axis2-version}</version>
</dependency>
<dependency>
    <groupId>org.apache.axis2</groupId>
    <artifactId>axis2-transport-http</artifactId>
    <version>${axis2-version}</version>
</dependency>
<dependency>
    <groupId>org.apache.axis2</groupId>
    <artifactId>axis2-transport-local</artifactId>
    <version>${axis2-version}</version>
</dependency>
<!-- Axis2客户端的包-->
<dependency>
    <groupId>org.apache.axis2</groupId>
    <artifactId>axis2-kernel</artifactId>
    <version>${axis2-version}</version>
</dependency>
<dependency>
    <groupId>org.apache.axis2</groupId>
    <artifactId>axis2-adb</artifactId>
    <version>${axis2-version}</version>
</dependency>
</dependencies>
```

### 编写服务接口

创建接口：

```java
public interface AxisHelloWorld {
    public String getMessage(String message);
}
```

实现类：

```java
@Service(value = "axisHelloWorldService")
public class AxisHelloWorldImpl implements AxisHelloWorld {
    @Override
    public String getMessage(String message) {
        return "---------Axis Server-------" + message;
    }
}
```

### 配置服务端

服务端的spring配置文件applicationContext.xml：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:aop="http://www.springframework.org/schema/aop"
       xmlns:context="http://www.springframework.org/schema/context"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-4.0.xsd
    http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context-4.0.xsd
    http://www.springframework.org/schema/aop http://www.springframework.org/schema/aop/spring-aop-4.0.xsd">

    <description>spring configuration</description>

    <!-- 扫描所有标注@Service的服务组件 -->
    <context:component-scan base-package="com.xncoding.axis2.service"/>

    <bean id="applicationContext" class="org.apache.axis2.extensions.spring.receivers.ApplicationContextHolder"/>

</beans>
```

然后再配置services.xml，这个是重点。

在/WEB-INF/目录下建立目录/services/axis/META-INF，然后再在这个目录下创建文件services.xml，内容如下：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<service name="axisHelloWorld">
    <parameter name="ServiceObjectSupplier" locked="false">
        org.apache.axis2.extensions.spring.receivers.SpringServletContextObjectSupplier
    </parameter>
    <parameter name="load-on-startup">true</parameter>
    <parameter name="SpringBeanName">axisHelloWorldService</parameter>
    <messageReceivers>
        <messageReceiver mep="http://www.w3.org/2004/08/wsdl/in-only"
                         class="org.apache.axis2.rpc.receivers.RPCInOnlyMessageReceiver"/>
        <messageReceiver mep="http://www.w3.org/2004/08/wsdl/in-out"
                         class="org.apache.axis2.rpc.receivers.RPCMessageReceiver"/>
    </messageReceivers>
</service>
```

再配置web.xml：

```xml

<web-app version="2.4"
         xmlns="http://java.sun.com/xml/ns/j2ee"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://java.sun.com/xml/ns/j2ee
	http://java.sun.com/xml/ns/j2ee/web-app_2_4.xsd">
    <description>Axis2实现的WebService服务器</description>
    <display-name>axis2-demo</display-name>
    <context-param>
        <param-name>contextConfigLocation</param-name>
        <param-value>classpath*:/applicationContext.xml</param-value>
    </context-param>
    <listener>
        <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
    </listener>

    <servlet>
        <servlet-name>AxisServlet</servlet-name>
        <servlet-class>org.apache.axis2.transport.http.AxisServlet</servlet-class>
        <load-on-startup>1</load-on-startup>
    </servlet>

    <servlet-mapping>
        <servlet-name>AxisServlet</servlet-name>
        <url-pattern>/services/*</url-pattern>
    </servlet-mapping>

    <welcome-file-list>
        <welcome-file>index.html</welcome-file>
        <welcome-file>index.jsp</welcome-file>
    </welcome-file-list>
</web-app>
```

### 启动测试

启动Spring MVC应用后，在浏览器中输入：<http://localhost:8080/services/axisHelloWorld?wsdl>

正确的返回WSDL文件，并且里面会有我定义的一个服务：

```xml

<wsdl:service name="axisHelloWorld">
```

服务器成功完成！

## 客户端

### 生成Stub类

Axis2提供了一个wsdl2java.bat命令可以根据WSDL文件自动产生调用WebService的代码。
wsdl2java.bat命令可以在<Axis2安装目录>/bin目录中找到。

在使用wsdl2java.bat命令之前需要设置AXIS2_HOME环境变量，该变量值是<Axis2安装目录>。然后再将%AXIS2_HOME%\bin目录加入PATH

然后再任意一个目录执行：

```
>wsdl2java -uri http://localhost:8080/services/axisHelloWorld?wsdl  -p com.xncoding.axis2.client -s -o stub
```

-p指定你生成的客户端Stub的包名

在执行完上面的命令后，就会发现在当前目录下多了个stub目录， 在stub/src/com/...目录可以找到一个xxxStub.java文件，
可通过该文件调用WebService，在程序中直接使用这个类。

### 编写测试方法

```java
@Test
public void test() throws Exception {
    //创建生成的stub类
    AxisHelloWorldStub stub = new AxisHelloWorldStub();
    //创建对应的方法类（这里我的webservice里有个getMessage的方法，具体的看你自己webservice里定义了何种方法）
    AxisHelloWorldStub.GetMessage msgParam = new AxisHelloWorldStub.GetMessage();
    msgParam.setMessage("消息消息");
    AxisHelloWorldStub.GetMessageResponse responseMsg = stub.getMessage(msgParam);
    //获取方法类的结果集
    String get_return = responseMsg.get_return();
    //打印输出结果
    System.out.println(get_return);
}
```

运行后输出结果：

````
---------Axis Server-------消息消息
````


