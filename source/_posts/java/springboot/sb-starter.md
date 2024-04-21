---
title: SpringBoot系列 - 自己写starter
date: 2017-07-22 16:22:19 +0800
toc: true
categories: [ java ]
tags: [ spring ]
---

Spring Boot由众多Starter组成，随着版本的推移Starter家族成员也与日俱增。在传统Maven项目中通常将一些层、组件拆分为模块来管理，
以便相互依赖复用，在Spring Boot项目中我们则可以创建自定义Spring Boot Starter来达成该目的。

可以认为starter是一种服务——使得使用某个功能的开发者不需要关注各种依赖库的处理，不需要具体的配置信息，
由Spring Boot自动通过classpath路径下的类发现需要的Bean，并织入相应的Bean。举个栗子，spring-boot-starter-jdbc这个starter的存在，
使得我们只需要在BookPubApplication下用@Autowired引入DataSource的bean就可以，Spring Boot会自动创建DataSource的实例。

本篇将通过一个简单的例子来演示如何编写自己的starter。
<!-- more -->

这里讲一下我们的Starter要实现的功能，很简单，提供一个Service，包含一个能够将字符串加上前后缀的方法String wrap(String word)。
而具体的前缀和后缀是通过读取SpringBoot配置文件`application.yml`而获取的。

## 添加maven依赖

第一步当然是创建一个maven工程，添加SpringBoot的自动配置的依赖：

```xml

<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.xncoding</groupId>
    <artifactId>simple-spring-boot-starter</artifactId>
    <version>1.0.0-SNAPSHOT</version>
    <packaging>jar</packaging>

    <name>simple-spring-boot-starter</name>
    <description>一个简单的自定义starter</description>
    <url>http://maven.apache.org</url>

    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <java.version>1.8</java.version>
    </properties>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.0.4.RELEASE</version>
    </parent>


    <dependencies>
        <!-- @ConfigurationProperties annotation processing (metadata for IDEs)
                 生成spring-configuration-metadata.json类，需要引入此类-->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-configuration-processor</artifactId>
            <optional>true</optional>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-autoconfigure</artifactId>
        </dependency>
    </dependencies>
</project>
```

注意其中 `spring-boot-configuration-processor` 的作用是编译时生成`spring-configuration-metadata.json`，
此文件主要给IDE使用，用于提示使用。如在intellij idea中，当配置此jar相关配置属性在`application.yml`，
你可以用ctlr+鼠标左键，IDE会跳转到你配置此属性的类中。

这里说下artifactId的命名问题，Spring 官方 Starter通常命名为`spring-boot-starter-{name}` 如 `spring-boot-starter-web`。

Spring官方建议非官方Starter命名应遵循`{name}-spring-boot-starter`的格式。

## 编写Service

```java
public class ExampleService {

    private String prefix;
    private String suffix;

    public ExampleService(String prefix, String suffix) {
        this.prefix = prefix;
        this.suffix = suffix;
    }
    public String wrap(String word) {
        return prefix + word + suffix;
    }
}
```

## 编写属性类

```java
@ConfigurationProperties("example.service")
public class ExampleServiceProperties {
    private String prefix;
    private String suffix;

    public String getPrefix() {
        return prefix;
    }

    public void setPrefix(String prefix) {
        this.prefix = prefix;
    }

    public String getSuffix() {
        return suffix;
    }

    public void setSuffix(String suffix) {
        this.suffix = suffix;
    }
}
```

## 编写自动配置类

```java
@Configuration
@ConditionalOnClass(ExampleService.class)
@EnableConfigurationProperties(ExampleServiceProperties.class)
public class ExampleAutoConfigure {

    private final ExampleServiceProperties properties;

    @Autowired
    public ExampleAutoConfigure(ExampleServiceProperties properties) {
        this.properties = properties;
    }

    @Bean
    @ConditionalOnMissingBean
    @ConditionalOnProperty(prefix = "example.service", value = "enabled",havingValue = "true")
    ExampleService exampleService (){
        return  new ExampleService(properties.getPrefix(),properties.getSuffix());
    }

}
```

解释下用到的几个和Starter相关的注解：

````
1. @ConditionalOnClass，当classpath下发现该类的情况下进行自动配置。
2. @ConditionalOnMissingBean，当Spring Context中不存在该Bean时。
3. @ConditionalOnProperty(prefix = "example.service",value = "enabled",havingValue = "true")，当配置文件中example.service.enabled=true时。
````

## 添加spring.factories

最后一步，在`resources/META-INF/`下创建`spring.factories`文件，内容供参考下面：

```
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
com.xncoding.starter.config.ExampleAutoConfigure
```

如果有多个自动配置类，用逗号分隔换行即可。

OK，完事，运行 `mvn:install` 打包安装，一个Spring Boot Starter便开发完成了。

## 测试

打包好了当然要测试一下看看了。另外创建一个SpringBoot工程，在maven中引入这个starter依赖，
然后在单元测试中引入这个Service看看效果。

```xml

<dependency>
    <groupId>com.xncoding</groupId>
    <artifactId>simple-spring-boot-starter</artifactId>
    <version>1.0.0-SNAPSHOT</version>
</dependency>
```

修改`application.yml`配置文件，添加如下内容：

```yaml
example.service:
  enabled: true
  prefix: ppp
  suffix: sss
```

测试类：

```java
@RunWith(SpringRunner.class)
@SpringBootTest
public class ApplicationTests {
    @Autowired
    private ExampleService exampleService;

    @Test
    public void testStarter() {
        System.out.println(exampleService.wrap("hello"));
    }
}
```

运行结果：

```
ppphellosss
```

完美！

## 总结

总结下Starter的工作原理:

1. Spring Boot在启动时扫描项目所依赖的JAR包，寻找包含`spring.factories`文件的JAR包
1. 根据`spring.factories`配置加载AutoConfigure类
1. 根据 `@Conditional` 注解的条件，进行自动配置并将Bean注入Spring Context

## GitHub源码

[springboot-starter](https://github.com/yidao620c/SpringBootBucket/tree/master/springboot-starter)

