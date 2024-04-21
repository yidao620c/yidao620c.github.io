---
title: SpringBoot系列 - cxf实现WebService
date: 2017-07-13 18:25:37 +0800
toc: true
categories: [ java ]
tags: [ spring ]
---

说起web service最近几年restful大行其道，大有取代传统soap web service的趋势，
但是一些特有或相对老旧的系统依然使用了传统的soap web service，例如银行、航空公司的机票查询接口等。
本篇主要讲解spring boot整合cxf发布webservice服务和spring boot整合cxf客户端调用webservice服务。
<!-- more -->

## maven依赖

```xml

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

    <!-- CXF webservice -->
    <dependency>
        <groupId>org.apache.cxf</groupId>
        <artifactId>cxf-spring-boot-starter-jaxws</artifactId>
        <version>3.2.4</version>
    </dependency>
    <!-- CXF webservice -->

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>org.hamcrest</groupId>
        <artifactId>hamcrest-all</artifactId>
        <version>1.3</version>
        <scope>test</scope>
    </dependency>
</dependencies>
```

## 服务接口

先定义一个简单的用户类User：

```java
package com.xncoding.webservice.model;

public class User {
    private Long id;
    private String name;
    private Integer age;

    public User() {
    }

    public User(Long id, String name, Integer age) {
        this.id = id;
        this.name = name;
        this.age = age;
    }

    // getter and setter
}
```

注意，上面的包名是下面接口定义的targetNamespace的倒序。

创建一个简单的服务接口，定义了两个方法，一个返回字符串，一个返回一个实体类User：

```java
package com.xncoding.webservice.service;

import com.xncoding.webservice.model.User;

import javax.jws.WebMethod;
import javax.jws.WebParam;
import javax.jws.WebService;

@WebService(name = "CommonService", // 暴露服务名称
        targetNamespace = "http://model.webservice.xncoding.com/"// 命名空间,一般是接口的包名倒序
)
public interface ICommonService {
    @WebMethod
    public String sayHello(@WebParam(name = "userName") String name);

    @WebMethod
    public User getUser(@WebParam(name = "userName") String name);
}
```

## 接口实现

接下来实现这个接口，编写相应的业务逻辑：

```java
package com.xncoding.webservice.service.impl;

import com.xncoding.webservice.model.User;
import com.xncoding.webservice.service.ICommonService;
import org.springframework.stereotype.Component;

import javax.jws.WebService;

@WebService(serviceName = "CommonService", // 与接口中指定的name一致
        targetNamespace = "http://model.webservice.xncoding.com/", // 与接口中的命名空间一致,一般是接口的包名倒
        endpointInterface = "com.xncoding.webservice.service.ICommonService"// 接口地址
)
@Component
public class CommonServiceImpl implements ICommonService {

    @Override
    public String sayHello(String name) {
        return "Hello ," + name;
    }

    @Override
    public User getUser(String name) {
        return new User(1000L, name, 23);
    }
}
```

## 配置类

接下来编写cxf的配置类：

```java
@Configuration
public class CxfConfig {
    @Autowired
    private Bus bus;

    @Autowired
    ICommonService commonService;

    /**
     * JAX-WS
     **/
    @Bean
    public Endpoint endpoint() {
        EndpointImpl endpoint = new EndpointImpl(bus, commonService);
        endpoint.publish("/CommonService");
        return endpoint;
    }
}
```

这里把`Commonservice`接口发布在了路径`/services/CommonService`
下，wsdl文档路径为`http://localhost:{port}/services/CommonService?wsdl`

如果你想自定义wsdl的访问url，那么可以在application.yml中自定义：

```yaml
cxf:
  path: /services  # 替换默认的/services路径
```

## 客户端访问

直接客户端去访问有两种方式

### 代理类工厂的方式

这种方式需要拿到对方的接口

```java
@Test
public void cl1() {
    try {
        // 接口地址
        // 代理工厂
        JaxWsProxyFactoryBean jaxWsProxyFactoryBean = new JaxWsProxyFactoryBean();
        // 设置代理地址
        jaxWsProxyFactoryBean.setAddress(wsdlAddress);
        // 设置接口类型
        jaxWsProxyFactoryBean.setServiceClass(ICommonService.class);
        // 创建一个代理接口实现
        ICommonService cs = (ICommonService) jaxWsProxyFactoryBean.create();
        // 数据准备
        String userName = "Leftso";
        // 调用代理接口的方法调用并返回结果
        String result = cs.sayHello(userName);
        System.out.println("返回结果:" + result);
    } catch (Exception e) {
        e.printStackTrace();
    }
}
```

### 动态调用方式

```java
@Test
public void cl2() {
    // 创建动态客户端
    JaxWsDynamicClientFactory dcf = JaxWsDynamicClientFactory.newInstance();
    Client client = dcf.createClient(wsdlAddress);
    // 需要密码的情况需要加上用户名和密码
    // client.getOutInterceptors().add(new ClientLoginInterceptor(USER_NAME, PASS_WORD));
    Object[] objects;
    try {
        // invoke("方法名",参数1,参数2,参数3....);
        objects = client.invoke("sayHello", "Leftso");
        System.out.println("返回类型：" + objects[0].getClass());
        System.out.println("返回数据:" + objects[0]);
    } catch (java.lang.Exception e) {
        e.printStackTrace();
    }
}

@Test
public void cl3() {
    // 创建动态客户端
    JaxWsDynamicClientFactory dcf = JaxWsDynamicClientFactory.newInstance();
    Client client = dcf.createClient(wsdlAddress);
    Object[] objects;
    try {
        // invoke("方法名",参数1,参数2,参数3....);
        objects = client.invoke("getUser", "张三");
        System.out.println("返回类型：" + objects[0].getClass());
        System.out.println("返回数据:" + objects[0]);
        User user = (User) objects[0];
        System.out.println("返回对象User.name=" + user.getName());
    } catch (java.lang.Exception e) {
        e.printStackTrace();
    }
}
```

注意到如果方法返回对象的话，那么对象所在的包一定是targetNamespace的倒序。

## 生成客户端代码

这是推荐做法，就是根据wsdl的访问路径来生成客户端代码。这里有两种生成方式

### Apache的wsdl2java工具

```
wsdl2java -autoNameResolution http://xxx?wsdl
```

### JDK自带的工具（推荐）

JDK自己内置了一个wsimport工具，这个是推荐的生成方式。

```
wsimport -encoding utf-8 -p com.xncoding.webservice.client -keep http://xxx?wsdl -s d:/ws -B-XautoNameResolution
```

其中：

```
-encoding ：指定编码格式（此处是utf-8的指定格式）
-keep：是否生成Java源文件
-s：指定.java文件的输出目录
-d：指定.class文件的输出目录
-p：定义生成类的包名，不定义的话有默认包名
-verbose：在控制台显示输出信息
-b：指定jaxws/jaxb绑定文件或额外的schemas
-extension：使用扩展来支持SOAP1.2
```

上面的命令会在`d:/ws`目录下面生成相应的客户端代码。

客户端使用例子：

```java
CommonService_Service c = new CommonService_Service();
com.xncoding.webservice.client.User user = c.getCommonServiceImplPort().getUser("Tom");
assertThat(user.getName(), is("Tom"));
```

## GitHub源码

[springboot-cxf](https://github.com/yidao620c/SpringBootBucket/tree/master/springboot-cxf)

