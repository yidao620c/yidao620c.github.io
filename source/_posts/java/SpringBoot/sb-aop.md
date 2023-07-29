---
title: SpringBoot系列 - 使用AOP
date: 2017-07-24 22:26:19 +0800
toc: true
categories: [ java ]
tags: [ spring ]
---

AOP（面向切面编程）是Spring的两大核心功能之一，功能非常强大，为解耦提供了非常优秀的解决方案。
现在就以springboot中aop的使用来了解一下如何使用aop。
写几个简单的Spring RESTful服务接口方法，实现方法前面或后面打印日志。
<!-- more -->

## AOP术语定义

Spring的AOP中有几个重要概念搞清楚就行

* 执行点（Executepoint） - 类初始化，方法调用。
* 连接点（Joinpoint） - 执行点+方位的组合，可确定Joinpoint，比如类开始初始化前，类初始化后，方法调用前，方法调用后。
* 切点（Pointcut） - 在众多执行点中，定位感兴趣的执行点。Executepoint相当于数据库表中的记录，而Pointcut相当于查询条件。
* 增强（Advice） - 织入到目标类连接点上的一段程序代码。除了一段程序代码外，还拥有执行点的方位信息。
* 目标对象（Target） - 增强逻辑的织入目标类
* 引介（Introduction） - 一种特殊的增强（advice），它为类添加一些额外的属性和方法，动态为业务类添加其他接口的实现逻辑，让业务类成为这个接口的实现类。
* 代理（Proxy） - 一个类被AOP织入后，产生一个结果类，它便是融合了原类和增强逻辑的代理类。
* 切面（Aspect） - 切面由切点（Pointcut）和增强（Advice/Introduction）组成，既包括横切逻辑定义，也包括连接点定义。

AOP工作重点：

1. 如何通过切点（Pointcut）和增强（Advice）定位到连接点（Jointpoint）上；
2. 如何在增强（Advice）中编写切面的代码。

## 添加maven依赖

```xml

<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>2.0.4.RELEASE</version>
</parent>

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
    <artifactId>spring-boot-starter-aop</artifactId>
</dependency>

<build>
    <plugins>
        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
            <executions>
            </executions>
        </plugin>
    </plugins>
</build>
</dependencies>
```

## 配置application.yml

```yaml
server.port: 8092
```

## 创建Controller

没有结果返回的示例：

```java
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

/**
 * Description:
 */
@RestController
public class UserController {
    @RequestMapping("/first")
    public Object first() {
        return "first controller";
    }

    @RequestMapping("/doError")
    public Object error() {
        return 1 / 0;
    }
}
```

## 创建切面类

```java
/**
 * 日志切面
 */
@Aspect
@Component
public class LogAspect {
    @Pointcut("execution(public * com.xncoding.aop.controller.*.*(..))")
    public void webLog(){}

    @Before("webLog()")
    public void deBefore(JoinPoint joinPoint) throws Throwable {
        // 接收到请求，记录请求内容
        ServletRequestAttributes attributes = (ServletRequestAttributes) RequestContextHolder.getRequestAttributes();
        HttpServletRequest request = attributes.getRequest();
        // 记录下请求内容
        System.out.println("URL : " + request.getRequestURL().toString());
        System.out.println("HTTP_METHOD : " + request.getMethod());
        System.out.println("IP : " + request.getRemoteAddr());
        System.out.println("CLASS_METHOD : " + joinPoint.getSignature().getDeclaringTypeName() + "." + joinPoint.getSignature().getName());
        System.out.println("ARGS : " + Arrays.toString(joinPoint.getArgs()));

    }

    @AfterReturning(returning = "ret", pointcut = "webLog()")
    public void doAfterReturning(Object ret) throws Throwable {
        // 处理完请求，返回内容
        System.out.println("方法的返回值 : " + ret);
    }

    //后置异常通知
    @AfterThrowing("webLog()")
    public void throwss(JoinPoint jp){
        System.out.println("方法异常时执行.....");
    }

    //后置最终通知,final增强，不管是抛出异常或者正常退出都会执行
    @After("webLog()")
    public void after(JoinPoint jp){
        System.out.println("方法最后执行.....");
    }

    //环绕通知,环绕增强，相当于MethodInterceptor
    @Around("webLog()")
    public Object arround(ProceedingJoinPoint pjp) {
        System.out.println("方法环绕start.....");
        try {
            Object o =  pjp.proceed();
            System.out.println("方法环绕proceed，结果是 :" + o);
            return o;
        } catch (Throwable e) {
            e.printStackTrace();
            return null;
        }
    }
}
```

## 测试效果

启动项目，执行如下mvn命令：

```
mvn spring-boot:run
```

模拟正常执行的情况，访问`http://localhost:8092/first`，看控制台结果：

```
方法环绕start.....
URL : http://localhost:8092/first
HTTP_METHOD : GET
IP : 0:0:0:0:0:0:0:1
CLASS_METHOD : com.xncoding.aop.controller.UserController.first
ARGS : []
方法环绕proceed，结果是 :first controller
方法最后执行.....
方法的返回值 : first controller
```

模拟出现异常时的情况，访问`http://localhost:8092/doError`，看控制台结果：

```
方法环绕start.....
URL : http://localhost:8092/doError
HTTP_METHOD : GET
IP : 0:0:0:0:0:0:0:1
CLASS_METHOD : com.xncoding.aop.controller.UserController.error
ARGS : []
java.lang.ArithmeticException: / by zero

...中间省略...

方法最后执行.....
方法的返回值 : null
```

通过上面的简单的例子，可以看到aop的执行顺序。知道了顺序后，就可以在相应的位置做切面处理了。

## 切面注解说明

* @Aspect 作用是把当前类标识为一个切面供容器读取
* @Pointcut 定义切点，切点方法不用任何代码，返回值是void，重要的是条件表达式
* @Before 标识一个前置增强方法，相当于BeforeAdvice的功能
* @AfterReturning 后置增强，相当于AfterReturningAdvice，方法退出时执行
* @AfterThrowing 异常抛出增强，相当于ThrowsAdvice
* @After final增强，不管是抛出异常或者正常退出都会执行
* @Around 环绕增强，相当于MethodInterceptor

方法参数说明：

除了@Around外，每个方法里都可以加或者不加参数JoinPoint。

JoinPoint里包含了类名、被切面的方法名，参数等属性，可供读取使用。

@Around参数必须为ProceedingJoinPoint，pjp.proceed相应于执行被切面的方法。

@AfterReturning方法里，可以加returning = "xxx"，xxx即为在controller里方法的返回值，本例中的返回值是"first controller"。

@AfterThrowing方法里，可以加throwing = "XXX"，读取异常信息，如本例中可以改为：

```java
    //后置异常通知
    @AfterThrowing(throwing = "ex", pointcut = "webLog()")
    public void throwss(JoinPoint jp, Exception ex){
        System.out.println("方法异常时执行.....");
    }
```

一般常用的有before和afterReturn组合，或者单独使用Around，即可获取方法开始前和结束后的切面。

## 关于切点PointCut

execution函数用于匹配方法执行的连接点，语法为：

```
execution(方法修饰符(可选)  返回类型  方法名  参数  异常模式(可选))
```

参数部分允许使用通配符：

1.
    * 匹配任意字符，但只能匹配一个元素
1. .. 匹配任意字符，可以匹配任意多个元素，表示类时，必须和*联合使用
1.
    + 必须跟在类名后面，如Horseman+，表示类本身和继承或扩展指定类的所有类

除了execution()，Spring中还支持其他多个函数，这里列出名称和简单介绍

1. @annotation() 表示标注了指定注解的目标类方法
1. args() 通过目标类方法的参数类型指定切点
1. @args() 通过目标类参数的对象类型是否标注了指定注解指定切点
1. within() 通过类名指定，它将匹配指定类型
1. target() 通过类名指定，同时包含所有子类
1. @within() 匹配标注了指定注解的类及其子类，必须是在目标对象上声明这个注解，在接口上声明的对它不起作用
1. @target() 匹配标注了指定注解的类，必须是在目标对象上声明这个注解，在接口上声明的对它不起作用

逻辑运算符

表达式可由多个切点函数通过逻辑运算组成，与(&&)、 或(||)、 非(!)

比如`execution(* chop(..)) && target(Horseman) 表示Horseman及其子类的chop方法`

## 自定义注解

一般多用于某些特定的功能，我们来自定义一个注解：

```java
@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
public @interface UserAccess {
    String desc() default "无信息";
}
```

在Controller里加个方法：

```java
@RequestMapping("/second")
@UserAccess(desc = "second")
public Object second() {
    return "second controller";
}
```

创建一个新的切面类：

```java

import org.aspectj.lang.JoinPoint;
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.*;
import org.springframework.stereotype.Component;

@Component
@Aspect
public class UserAccessAspect {

    @Pointcut(value = "@annotation(com.xncoding.aop.aspect.UserAccess)")
    public void access() {

    }

    @Before("access()")
    public void deBefore(JoinPoint joinPoint) throws Throwable {
        System.out.println("second before");
    }

    @Around("@annotation(userAccess)")
    public Object around(ProceedingJoinPoint pjp, UserAccess userAccess) {
        //获取注解里的值
        System.out.println("second around:" + userAccess.desc());
        try {
            return pjp.proceed();
        } catch (Throwable throwable) {
            throwable.printStackTrace();
            return null;
        }
    }
}
```

主要看一下@Around注解这里，如果需要获取在controller注解中赋给UserAccess的desc里的值，就需要这种写法，这样UserAccess参数就有值了。

执行：`mvn spring-boot:run`

启动项目，访问`http://localhost:8092/second`，看控制台：

```
方法环绕start.....
URL : http://localhost:8092/second
HTTP_METHOD : GET
IP : 0:0:0:0:0:0:0:1
CLASS_METHOD : com.xncoding.aop.controller.UserController.second
ARGS : []
second around:second
second before
方法环绕proceed，结果是 :second controller
方法最后执行.....
方法的返回值 : second controller
```

通知结果可以看到，两个aop切面类都工作了，顺序呢就是下面的

![](https://xnstatic-1253397658.file.myqcloud.com/aop20.png)

spring
aop就是一个同心圆，要执行的方法为圆心，最外层的order最小。从最外层按照AOP1、AOP2的顺序依次执行doAround方法，doBefore方法。然后执行method方法，最后按照AOP2、AOP1的顺序依次执行doAfter、doAfterReturn方法。也就是说对多个AOP来说，先before的，一定后after。

对于上面的例子就是，先外层的就是对所有controller的切面，内层就是自定义注解的。
那不同的切面，顺序怎么决定呢，尤其是同格式的切面处理，譬如两个execution的情况，那spring就是随机决定哪个在外哪个在内了。

所以大部分情况下，我们需要指定顺序，最简单的方式就是在Aspect切面类上加上@Order(1)
注解即可，order越小最先执行，也就是位于最外层。像一些全局处理的就可以把order设小一点，具体到某个细节的就设大一点。

## GitHub源码

[springboot-aop](https://github.com/yidao620c/SpringBootBucket/tree/master/springboot-aop)

*更新：2018年8月升级至springboot 2*

