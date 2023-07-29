---
title: 利用httpd对Tomcat进行负载均衡
date: 2015-10-13 10:59:15 +0800
toc: true
categories: [ 中间件 ]
tags: [ tomcat ]
---

操作系统：CentOS 6.5_x86_64

前提：提前准备好编译环境，防火墙和selinux都关闭

主机IP：两台机器，192.168.203.103、192.168.203.104

安装软件：jdk-8u51-linux-x64, apache-tomcat-8.0.24, tomcat-connectors-1.2.41, httpd-2.2.15, httpd-devel-2.2.15
<!-- more -->

### 一、两台机器都安装JAVA8

```bash
sudo rpm -qa | grep jdk
jdk-1.7.0_45-fcs.x86_64
sudo rpm -e jdk-1.7.0_45
```

下载JDK8的包

```bash
wget --no-cookies --no-check-certificate --header "Cookie: gpw_e24=http%3A%2F%2Fwww.oracle.com%2F; oraclelicense=accept-securebackup-cookie" "http://download.oracle.com/otn-pub/java/jdk/8u60-b27/jdk-8u60-linux-x64.tar.gz"
```

如果上述链接失效，请去官网下载最新的源码包。

```bash
cd /opt/
tar xzf jdk-8u51-linux-x64.tar.gz
cd /opt/jdk1.8.0_51/
sudo chown -R root:root /opt/jdk1.8.0_51/
sudo alternatives --install /usr/bin/java java /opt/jdk1.8.0_51/bin/java 2
sudo alternatives --config java
```

得到以下输出，选择刚刚安装的jdk8即可：

```
There are 3 programs which provide 'java'.

  Selection    Command
-----------------------------------------------
*  1           /opt/jdk1.7.0_71/bin/java
 + 2           /opt/jdk1.8.0_25/bin/java
   3           /opt/jdk1.8.0_51/bin/java

Enter to keep the current selection[+], or type selection number: 3
```

然后再配置下javac和jar

```bash
sudo alternatives --install /usr/bin/jar jar /opt/jdk1.8.0_51/bin/jar 2
sudo alternatives --install /usr/bin/javac javac /opt/jdk1.8.0_51/bin/javac 2
sudo alternatives --set jar /opt/jdk1.8.0_51/bin/jar
sudo alternatives --set javac /opt/jdk1.8.0_51/bin/javac
```

查看下JDK版本

```bash
java -version
```

修改环境变量

```bash
sudo vim /etc/profile
```

输入以下内容

```bash
export JAVA_HOME=/opt/jdk1.8.0_51
export JRE_HOME=/opt/jdk1.8.0_51/jre
export PATH=$PATH:$JAVA_HOME/bin
```

执行

```bash
source /etc/profile
```

### 二、两台机器安装tomcat

1.下载安装tomcat

```bash
wget http://mirrors.cnnic.cn/apache/tomcat/tomcat-8/v8.0.24/bin/apache-tomcat-8.0.24.tar.gz
tar xf apache-tomcat-8.0.24.tar.gz -C /usr/local/
cd /usr/local/
ln -sv apache-tomcat-8.0.24 tomcat
```

2.配置环境变量

```bash
vim /etc/profile.d/tomcat.sh
```

```bash
CATALINA_BASE=/usr/local/tomcat
PATH=$CATALINA_BASE/bin:$PATH
export PATH CATALINA_BASE
```

执行：

```bash
. /etc/profile.d/tomcat.sh
```

3.查看状态：

```bash
catalina.sh version
```

4.提供启动脚本

sudo vim /etc/init.d/tomcat

```bash

#!/bin/sh
# Tomcat init script for linux
# chkconfig: 2345 96 14
# description: The Apache Tomcat servlet/JSP container
# JAVA_OPTS='-Xms64m -Xmx128m'
JAVA_HOME=/opt/jdk
CATALINA_HOME=/usr/local/tomcat
export JAVA_HOME CATALINA_HOME

case $1 in
start)
  exec $CATALINA_HOME/bin/catalina.sh start ;;
stop)
  exec $CATALINA_HOME/bin/catalina.sh stop ;;
restart)
  echo "stoping tomcat ..."
  ps aux |grep tomcat/bin |grep -v "grep tomcat/bin" |while read line
  do
    linewords=($line)
    pid="${linewords[1]}"
    kill -9 $pid
  done
  # $CATALINA_HOME/bin/catalina.sh stop
  sleep 2
  echo "starting tomcat ..."
  exec $CATALINA_HOME/bin/catalina.sh start ;;
*)
  echo "Usage: $0 {start|stop|restart}"
  exit 1
  ;;
esac

```

执行：

```bash

sudo chmod +x /etc/init.d/tomcat
sudo chkconfig --add tomcat

```

5.编辑tomcat配置文件，只添加jvmRoute参数：

在第一台机子上面：

```bash
sudo vim /usr/local/tomcat/conf/server.xml
```

修改下面这句：

```
<Engine name="Catalina" defaultHost="localhost" jvmRoute="TomcatA">
```

在第二台机子上面：

```bash
sudo vim /usr/local/tomcat/conf/server.xml
```

修改下面这句：

```
<Engine name="Catalina" defaultHost="localhost" jvmRoute="TomcatB">
```

6.提供测试页面

第一台机器上：

```bash
sudo mkdir -pv /usr/local/tomcat/webapps/test/WEB-INF/{classes,lib}
sudo vim /usr/local/tomcat/webapps/test/index.jsp
```

写一个简单的JSP页面：

```html
<%@ page language="java" %>
<%@ page import="java.util.*" %>
<html>
    <head>
        <title>test</title>
    </head>
    <body>
        <%
            out.println("This is TomcatA");
        %>
    </body>
</html>
```

然后启动tomcat

```bash
sudo service tomcat start
```

这时候可以通过访问 `http://192.168.203.103:8080/test` 访问到这个页面

第二台机器上：

```bash
sudo mkdir -pv /usr/local/tomcat/webapps/test/WEB-INF/{classes,lib}
sudo vim /usr/local/tomcat/webapps/test/index.jsp
```

写一个简单的JSP页面：

```html
<%@ page language="java" %>
<%@ page import="java.util.*" %>
<html>
    <head>
        <title>test</title>
    </head>
    <body>
        <%
            out.println("This is TomcatB");
        %>
    </body>
</html>
```

然后启动tomcat

```bash
sudo service tomcat start
```

这时候可以通过访问`http://192.168.203.104:8080/test`访问到这个页面

### 三、利用mod_jk模块对tomcat进行负载均衡

利用httpd反向代理tomcat时有两种方法，分别要用到mod_proxy和mod_jk这两个模块。
mod_jk需要额外编译安装，不过它功能更强大，所以推荐mod_jk。
此模块只需要在一台机器上安装，我们这里在第一台机器（103）上安装。

1.安装httpd：

```
yum -y install httpd httpd-devel
```

2.安装mod_jk.so模块：

```bash
wget http://mirrors.cnnic.cn/apache/tomcat/tomcat-connectors/jk/tomcat-connectors-1.2.41-src.tar.gz
tar xf tomcat-connectors-1.2.41-src.tar.gz
cd tomcat-connectors-1.2.41-src/native/
./configure --with-apxs=/usr/sbin/apxs
sudo make && sudo make install
```

3.提供额外的httpd模块配置文件：

```bash
vim /etc/httpd/conf.d/httpd-jk.conf
```

```
# Load the mod_jk
LoadModule  jk_module  modules/mod_jk.so
JkWorkersFile  /etc/httpd/conf.d/workers.properties
JkLogFile  logs/mod_jk.log
JkLogLevel  debug
JkMount  /*  lb1
JkMount  /status/  stat1
```

4.配置mod_jk模块的配置文件workers.properties：

```bash
vim /etc/httpd/conf.d/workers.properties
```

```
worker.list = lb1,stat1
worker.TomcatA.type = ajp13
worker.TomcatA.host = 192.168.203.103
worker.TomcatA.port = 8009
worker.TomcatA.lbfactor = 1
worker.TomcatB.type = ajp13
worker.TomcatB.host = 192.168.203.104
worker.TomcatB.port = 8009
worker.TomcatB.lbfactor = 1
worker.lb1.type = lb
worker.lb1.sticky_session = 0
worker.lb1.balance_workers = TomcatA, TomcatB
worker.stat1.type = status
```

5.启动httpd测试：
我们先去修改下hostname，还有httpd的domainname，`sudo vim /etc/hosts`

```
127.0.0.1	localhost centos03
```

然后修改httpd的配置文件，`sudo vim /etc/httpd/conf/httpd.conf`
修改这一行：

```
ServerName localhost:80
```

最后我们启动httpd服务：

```bash
sudo service httpd start
```

用浏览器打开`http://192.168.203.103/test`，我们不断刷新，可以看到效果。

6.修改httpd默认端口号方法

```bash
sudo vim /etc/httpd/conf/httpd.conf
```

修改两个地方

```
#Listen 12.34.56.78:80
Listen 80
#把80改为你设置的端口，我设置端口为8088

Listen 8088

NameVirtualHost *:80
#把80改为你设置的端口，我设置端口为8088
NameVirtualHost *:8088
```

保存修改，退出，重启httpd服务。

