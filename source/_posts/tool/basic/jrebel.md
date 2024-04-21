---
title: IDEA集成JRebel热部署和远程调试
date: 2017-10-29 18:09:22 +0800
toc: true
categories: [ 开发工具 ]
tags: [ 开发工具 ]
---

在Java Web开发中，一般更新了Java文件后要手动重启Tomcat服务器才能生效，浪费不少生命啊，
自从有了JRebel这神器的出现，不论是更新类还是更新Spring配置文件都能做到立马生效，大大提高开发效率。

JRebel的使用方式最常见还是通过插件方式使用，这里我介绍下在IntelliJ IDEA中怎样集成JRebel，
另外还顺便介绍一下IDEA如何进行远程调试。
<!-- more -->

## 安装

IDEA里面安装插件比较简单，File --> setttings --> Plugins,找到`Browe Repositories`按钮，
查找需要的JRebel插件，点击Install即可。

![](https://xnstatic-1253397658.file.myqcloud.com/jrebel01.png)

安装完插件后重启IDEA即可看到JRebel的图标了，绿色的小火箭。

## 激活

### 通过myjrebel

下一步就是激活JRebel了，现在 JRebel 对个人非商业用途的用户永久免费，只需要分享一下使用统计。
访问：https://my.jrebel.com/ 使用 Facebook 或者 Twitter 帐号登录获取永久激活码。
然后注册完，在如下页面就有注册码：

![](https://xnstatic-1253397658.file.myqcloud.com/jrebel02.png)

获取到注册码后复制下来，然后点击 Help --> JRebel --> Activation

![](https://xnstatic-1253397658.file.myqcloud.com/jrebel03.png)

输入激活码即可：

![](https://xnstatic-1253397658.file.myqcloud.com/jrebel04.png)

### 通过LicenseServer

在2018/07/05这天IDEA突然提示JRebel的激活码不能用了，然后打开`https://zeroturnaround.com/software/jrebel/myjrebel-discontinued/`一看，

```
We’ve made a difficult decision. As of July 5, myJRebel is no longer available.
But don’t worry! All myJRebel users active within the last 12 months are eligible to continue using all JRebel’s wonderful 
features by moving to JRebel commercial license at a massively discounted price: $99 (over 80% off the regular price).
```

通过开源社区找到一个很好用的LicenseServer，通过这个可以，gitee地址：

```
https://gitee.com/gsls200808/JrebelLicenseServerforJava
```

Packing a runnable jar:

```bash
mvn package
java -jar JrebelBrainsLicenseServerforJava-1.0-SNAPSHOT-jar-with-dependencies.jar -p 8082
```

default port is 8081.

然后在IDEA里面的JRebel激活中输入通过licensing service：

```
http://ip:8082/fdf12095-3e08-49a0-92ee-5b459d5431dd
```

email也随便填一个，激活即可。

## 配置JRebel插件

安装完成后，简单的配置就可以使用Jrebel的强大功能，在IntelliJ左下角，选择JRebel选项卡，将第一个勾上即可

![](https://xnstatic-1253397658.file.myqcloud.com/jrebel05.png)

如果是Web项目，并且你使用Tomcat Web容器来开发的话，还需要配置运行项目，
点击 Run -> Edit Configurations...，在Tomcat配置里面，设置`On 'Update'' action` 和 `On frame deactivation`。

注意：如果web启动的时候，出现内存溢出现象则需要配置一下VM options：

![](https://xnstatic-1253397658.file.myqcloud.com/jrebel06.png)

但如果你用Jetty容器，那就不用像上面这样配置了。

接下来就直接点击绿色小火箭，运行/调试都可以。

## 远程调试

准备工作

* 明确远程服务器的IP地址，比如我是：123.207.66.156
* 关掉服务器防火墙：`systemctl stop firewalld.service`

在Run/Configuration里面，新增一个Remote Server配置。

复制 Remote Server 自动生成的 JVM 参数，等下有用，如下图，比如我的是：

```
-agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=5005
```

在Host里面修改成服务器IP地址，比如我的是123.207.66.156

![](https://xnstatic-1253397658.file.myqcloud.com/jrebel09.png)

### 远程Tomcat配置

编辑文件`catalina.sh`，在最上面添加：

```bash
export JAVA_OPTS='-agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=5005'
```

如果你的项目有特殊 JVM 参数，那你就把你的那部分参数和这部分参数合并在一起。

### 远程Jetty

jetty 不像Tomcat那样需要安装，只要有jetty的jar包就可以启动我们想要启动的应用，启动命令如下：

```bash
nohup java -agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=5005 -jar ${jarfile} >/dev/null 2>&1 &
```

### 开始调试

1. 启动服务器 Tomcat/Jetty
2. 启动本地 Remote Server
3. 如果可以看到如下图效果，表示已经连接成功了，接下里就是跟往常一样，
   在本地代码上设置断点，然后你访问远程的地址，触发到该代码自动就会在本地停住。

![](https://xnstatic-1253397658.file.myqcloud.com/jrebel10.png)

