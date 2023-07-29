---
title: OSGi简易教程
date: 2016-04-16 01:02:42 +0800
toc: true
categories: [ java ]
tags: [ java ]
---

开篇我先解释一下两个容易搞混的术语：

* OSGi: The Open Services Gateway Initiative(开放服务网关协议)
  ，是用来模块化开发和部署Java应用程序和库（library）的一种架构，已经成为Java模块化事实上的标准了。详细解释请参考[OSGi简介](http://www.osgi.com.cn/article/7289226)
* WSGI: Web Server Gateway Interface(Web服务器网关接口)
  ，具体的来说，WSGI是一个Python专用规范，定义了Web服务器如何与Python应用程序进行交互，使得使用Python写的Web应用程序可以和Web服务器对接起来。最新版本在Python的PEP-3333定义的。详细解释请参考[WSGI简介](https://segmentfault.com/a/1190000003069785)

所以，弄清楚之后才发现只有一个字母的差别，但是这两个东东一毛钱的关系都没有，^_^

跟Java的Servlet协议和EJB协议类似，OSGi协议定义了两件事：一组OSGi容器必须实现的服务，以及容器和你的应用程序之间的接口标准。
也许你会搞不懂为什么我们已经有了Servelt容器来构建Web应用程序，有了EJB容器来构建企业级事务应用程序，那我们还要这个OSGi容器干嘛？
简单来讲就是OSGi容器是特地用来开发非常复杂的大型Java应用程序的，你需要将系统分解成很多个模块。
<!-- more -->

### 使用Equinox构建一个小应用

OSGi容器的开源实现现在主要有三种：Knopflerfish, Equinox 和 Apache Felix。

接下来我通过例子来演示如何使用Eclipse OSGi容器Equinox来构建一个helloworld应用。同时会介绍到OSGi中的ServiceFactory和ServiceTracker类。

在OSGi里面，各个模块都是以bundle的形式存在。每个bundle包含各自的java类和其他资源，并可以为其他的bundle提供服务。

#### 创建bundle

打开eclipse，通过下面的步骤来创建一个Hello World的bundle

1. 菜单选择File --> New --> Project，打开创建工程对话框。
2. 选择`Plug-in Project`点下一步
3. 输入如下的内容

- Project Name: com.javaworld.sample.HelloWorld
- Target Platform: OSGi framework --> Standard

4. 其他默认点下一步进入Plug-in Context对话框，还是默认点下一步
5. 这一步Template对话框里面选择Hello OSGi Bundle，然后完成

#### Activator.java

打开Activator.java这个类，应该类似下面这样：

```java
package com.javaworld.sample.helloworld;
import org.osgi.framework.BundleActivator;
import org.osgi.framework.BundleContext;
public class Activator implements BundleActivator {
    public void start(BundleContext context) throws Exception {
        System.out.println("Hello world");
    }
    public void stop(BundleContext context) throws Exception {
        System.out.println("Goodbye World");
    }
}
```

#### MANIFEST.MF

这个文件比较核心，它定义了你怎样去部署这个bundle。应该会类似下面这样。*注：#开头的是我自己加的注释*

```
# 这个是MF版本
Manifest-Version: 1.0
# 定义了Bundle遵循的OSGi规范版本。默认值为1，表示R3的Bundle，2表示R4及其后发布的版本。
Bundle-ManifestVersion: 2
# 该属性头为本Bundle定义了一个简短的、可以阅读的名称
Bundle-Name: HelloWorld Plug-in
# 提供了Bundle的一个全局的惟一的标志符，名称应该是基于反域名解析的。这个标记是必须的。
Bundle-SymbolicName: com.javaworld.sample.HelloWorld
# 该属性头给出了本Bundle的版本号
Bundle-Version: 1.0.0
# 该属性头给出了本Bundle中使用的监听器类名字，这个属性值是可选的。监听器将对Activator中的start()和stop()方法监听
Bundle-Activator: com.javaworld.sample.helloworld.Activator
# 该属性头是对本Bundle发行商的表述
Bundle-Vendor: JAVAWORLD
# Bundle运行所依赖的环境
Bundle-RequiredExecutionEnvironment: JavaSE-1.8
# 定义bundle的导入包
Import-Package: org.osgi.framework;version="1.3.0"
```

现在一切准备就绪了，你可以直接运行这个bundle。点击"Run Configuration"，新加一个OSGi Framework，把需要的bundle勾上，Run

注意，请在Target Platform里面把下面这些依赖添加进去
![](https://xnstatic-1253397658.file.myqcloud.com/20160512170036.png)

运行之后会在控制台打印hello world，那么说明执行成功了。

### OSGi控制台

OSGi控制台是OSGi容器的命令行接口，它可以让你启动、停止、安装、更新和卸载bundle。
在Eclipse下面的Open Console标签点击"Host OSGi Console"

下面是常见的几个命令：

* ss 显示所有已经安装了的bundle列表，包括bundle ID、简短名、状态
* start <bundleid> 启动一个bundle
* stop <bundleid> 停止一个bundle
* update <bundleid> 使用一个新的jar来更新bundle
* install <bundleURL> 在OSGi容器中安装一个新的bundle
* uninstall <bundleid> 从OSGi容器中卸载一个bundle
* exit 退出osgi环境
* refresh <bundleid>  刷新对应id的Bundle
* ls 显示所有services 如果没有发布service会显示找不到该命令
* ls –c <bundleid> 显示该bundle中所有发布service的详细信息，对与unsatisfied的component查错很有用

这些都是标准的OSGi命令，所以你可以适用于任何OSGi容器

### 依赖管理

OSGi协议允许你将应用切分为多个子模块，然后各自管理自己的依赖，默认情况下bundle里的东西对外是不可见的。要想让别的bundle可以调用到内部服务需要导出相应的包，同时其他想要使用的bundle还要导入此包。

#### 导出一个包

我们再创建一个`com.javaworld.sample.HelloService`的bundle，然后将其导出。这个bundle构建过程跟前面步骤一样，这里不再重复。

然后创建好后我们在HelloService这个bundle里面新建一个`com.javaworld.sample.service.HelloService.java`接口

```java
public interface HelloService {
    public String sayHello();
}
```

再创建一个实现了这个接口的类`com.javaworld.sample.service.impl.HelloServiceImpl.java`

```java
public class HelloServiceImpl implements HelloService{
    public String sayHello() {
        System.out.println("Inside HelloServiceImple.sayHello()");
        return "Say Hello";
    }
}
```

打开HelloService中的`MANIFEST.MF`文件，添加Export-Package选项

```
Export-Package: com.javaworld.sample.service
```

#### 导入一个包

在HelloWorld bundle里面，打开它的`MANIFEST.MF`文件，添加Import-Package

```
Import-Package: com.javaworld.sample.service,
 org.osgi.framework;version="1.3.0"
```

打开`com.javaworld.sample.helloworld.Activator.java`源文件，现在你是可以访问HelloServiceImpl类了，
可编译通过但是直接调用会出错。原因是OSGi特有的类加载机制

#### OSGi服务

OSGi很适合用来开发面向服务的应用程序，因为它会把接口和实现分开，调用者不需要知道任何服务实现细节，
而服务提供者可以任意修改服务实现而不用告知被调用者，只要保证接口不变即可。

#### 导出服务

在服务bundle里面新建一个`com.javaworld.sample.service.impl.HelloServiceActivator.java`

```java
public class HelloServiceActivator implements BundleActivator  {
    ServiceRegistration helloServiceRegistration;
    public void start(BundleContext context) throws Exception {
        HelloService helloService = new HelloServiceImpl();
        helloServiceRegistration =context.registerService(HelloService.class.getName(), helloService, null);
    }
    public void stop(BundleContext context) throws Exception {
        helloServiceRegistration.unregister();
    }
}
```

通过使用`BundleContext.registerService()`来导出服务，三个参数含义

* 第一个是服务的名字
* 第二个是服务的具体实现
* 第三个是服务的属性，是一个`Dictionary`对象

然后你要更改服务bundle的`MANIFEST.MF`文件，指定`HelloServiceActivator`为它的activator类

```
Bundle-Activator: com.javaworld.sample.service.impl.HelloServiceActivator
```

#### 导入服务

这一步我们修改HelloWorld的程序，开始导入我们刚刚创建的服务并实际使用它。修改`Activator`类

```java
public class Activator implements BundleActivator {
    ServiceReference helloServiceReference;
    public void start(BundleContext context) throws Exception {
        System.out.println("Hello World!!");
        helloServiceReference= context.getServiceReference(HelloService.class.getName());
        HelloService helloService =(HelloService)context.getService(helloServiceReference);
        System.out.println(helloService.sayHello());

    }
    public void stop(BundleContext context) throws Exception {
        System.out.println("Goodbye World!!");
        context.ungetService(helloServiceReference);
    }
}
```

`BundleContext.getServiceReference()`方法返回一个`ServiceReference`对象，
然后你可以调用`BundleContext.getService()`传入这个对象后获取真正的服务对象了。
再次运行这个HelloWorld看看，打印出`Inside HelloServiceImple.sayHello()`，说明已经成功调用了。

### 创建一个服务工厂

上一节我们在服务bundle里面创建了一个`HelloServiceImpl`对象，任何其他bundle导入的时候都会返回这个同样的对象。要是我们想对每个bundle返回不同对象要在呢么做呢？
解决方法是创建一个实现了`ServiceFactory`接口的类，然后注册这个工厂类的对象，而不是实际的服务对象。

```java
public class HelloServiceFactory implements ServiceFactory{
    private int usageCounter = 0;
    public Object getService(Bundle bundle, ServiceRegistration registration) {
        System.out.println("Create object of HelloService for " + bundle.getSymbolicName());
        usageCounter++;
        System.out.println("Number of bundles using service " + usageCounter);
        HelloService helloService = new HelloServiceImpl();
        return helloService;
    }
    public void ungetService(Bundle bundle, ServiceRegistration registration, Object service) {
        System.out.println("Release object of HelloService for " + bundle.getSymbolicName());
        usageCounter--;
        System.out.println("Number of bundles using service " + usageCounter);
    }
}
```

修改`HelloServiceActivator.java`中的`start()`方法

```java
public class HelloServiceActivator implements BundleActivator  {
    ServiceRegistration helloServiceRegistration;
    public void start(BundleContext context) throws Exception {
        HelloServiceFactory helloServiceFactory = new HelloServiceFactory();
        helloServiceRegistration =context.registerService(HelloService.class.getName(), helloServiceFactory, null);
    }
    public void stop(BundleContext context) throws Exception {
        helloServiceRegistration.unregister();
    }
}
```

再次运行也是同样的结果，不过这次效果是每个bundle请求会获取yield新的service对象了。

### 跟踪服务

有时候你需要跟踪某个服务什么时候被注册，被卸载之类的。这时候你可以使用`ServiceTracker`类，我们在HelloWorld里面新建一个类

```java HelloServiceTracker.java
public class HelloServiceTracker extends ServiceTracker {
    public HelloServiceTracker(BundleContext context) {
        super(context, HelloService.class.getName(),null);
    }
    public Object addingService(ServiceReference reference) {
        System.out.println("Inside HelloServiceTracker.addingService " + reference.getBundle());
        return super.addingService(reference);
    }
    public void removedService(ServiceReference reference, Object service) {
        System.out.println("Inside HelloServiceTracker.removedService " + reference.getBundle());
        super.removedService(reference, service);
    }
}
```

同时我们要修改` MANIFEST.MF`文件，导入包`org.osgi.util.tracker`

```
Import-Package: com.javaworld.sample.service,
 org.osgi.framework;version="1.3.0",
 org.osgi.util.tracker
```

然后修改`Activator.java`，让它使用这个`HelloServiceTracker`来获取真正的Service类

```java
public class Activator implements BundleActivator {
    HelloServiceTracker helloServiceTracker;
    public void start(BundleContext context) throws Exception {
        System.out.println("Hello World!!");
        helloServiceTracker= new HelloServiceTracker(context);
        helloServiceTracker.open();
        HelloService helloService = (HelloService)helloServiceTracker.getService();
        System.out.println(helloService.sayHello());

    }
    public void stop(BundleContext context) throws Exception {
        System.out.println("Goodbye World!!");
        helloServiceTracker.close();
    }
}
```

再次执行时，你会发现当你启动、停止这个HelloService bundle的时候，相应的跟踪器里面的方法也会执行。

### 总结

我们通过一个小的例子演示了一个使用OSGi框架开发模块化程序的最基础的东西，你已经知道怎样导入/导出包和服务了。如果想更深入学习，可以去[OSGi中文社区](http://www.osgi.com.cn/)
，那里有很多好的资源。

