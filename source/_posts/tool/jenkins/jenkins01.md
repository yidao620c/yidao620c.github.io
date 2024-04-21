---
title: Jenkins持续集成 - 安装配置
date: 2017-03-20 12:55:12 +0800
toc: true
categories: [ 开发工具 ]
tags: [ jenkins ]
---

Jenkins是一个用Java编写的开源的持续集成工具，前身是Hudson项目。
在与Oracle发生争执后，项目从Hudson复制过来继续发展。

Jenkins提供了软件开发的持续集成服务。它运行在Servlet容器中（例如Apache Tomcat）。
它支持许多软件配置管理（SCM）工具，可以执行基于Apache Ant和Apache Maven的项目，
以及任意的Shell脚本和Windows批处理命令。Jenkins的主要开发者是川口耕介，MIT许可证。
<!-- more -->

## 安装

环境：CentOS7.2、JDK8、Git2、Maven3、Nginx、Jenkins2

Jenkins有很多中安装方式，这里在CentOS7.2系统上面，我选择通过yum的方式安装，然后使用nginx做反向代理。

### 安装JDK8

先查查看系统上面是否有其他的旧版本，有的话就卸载掉：

```bash
sudo rpm -qa | grep jdk
jdk-1.7.0_45-fcs.x86_64
sudo rpm -e jdk-1.7.0_45
```

下载最新的JDK8压缩包

```
cd /opt/
wget --no-cookies --no-check-certificate --header "Cookie: gpw_e24=http%3A%2F%2Fwww.oracle.com%2F; oraclelicense=accept-securebackup-cookie" "http://download.oracle.com/otn-pub/java/jdk/8u121-b13/e9e7ea248e2c4826b92b3f075a80e441/jdk-8u121-linux-x64.tar.gz"
# or go to http://mirrors.linuxeye.com/jdk/
tar xzf jdk-8u121-linux-x64.tar.gz
```

使用alternatives命令安装java

```
cd /opt/jdk1.8.0_121/
sudo chown -R root:root /opt/jdk1.8.0_121/
alternatives --install /usr/bin/java java /opt/jdk1.8.0_121/bin/java 2
alternatives --config java
There are 3 programs which provide 'java'.

  Selection    Command
-----------------------------------------------
*  1           /opt/jdk1.7.0_71/bin/java
 + 2           /opt/jdk1.8.0_60/bin/java
   3           /opt/jdk1.8.0_121/bin/java

Enter to keep the current selection[+], or type selection number: 3
```

配置javac和jar命令

```
alternatives --install /usr/bin/jar jar /opt/jdk1.8.0_121/bin/jar 2
alternatives --install /usr/bin/javac javac /opt/jdk1.8.0_121/bin/javac 2
alternatives --set jar /opt/jdk1.8.0_121/bin/jar
alternatives --set javac /opt/jdk1.8.0_121/bin/javac
```

检查是否安装成功:

```
java -version

java version "1.8.0_121"
Java(TM) SE Runtime Environment (build 1.8.0_121-b13)
Java HotSpot(TM) 64-Bit Server VM (build 25.121-b13, mixed mode)
```

配置JAVA_HOME环境变量，编辑`/etc/profile`文件，最后加入

```
export JAVA_HOME=/opt/jdk1.8.0_121
export JRE_HOME=/opt/jdk1.8.0_121/jre
export PATH=$PATH:$JAVA_HOME/bin
```

`source /etc/profile` 搞定！

还需要安装git、maven，保证最后有git和mvn命令，这里省略。

### 安装Jenkins2

<https://pkg.jenkins.io/redhat-stable/>

```
sudo wget -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/redhat-stable/jenkins.repo
sudo rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io.key
sudo yum install jenkins
```

安装后产生下面3个目录文件

```
/etc/init.d/jenkins  ==>启动脚本
/usr/lib/jenkins/jenkins.war  ==>jenkins主程序包
/var/lib/jenkis/  ==>jenkins运行时环境配置，以及数据文件。刚安装好时为空。
```

### 迁移 Jenkins

复制老机器上的 `/var/lib/jenkins/` 的内容到新机器对应目录。

如果`/var/lib/jenkins/`目录实在太大，可以将各个项目的历史构建任务先删除了再copy:

```
rm -f /var/lib/jenkins/jobs/{projectname}/builds/*
rm -f /var/lib/jenkins/jobs/{projectname}/modules/*
```

### 更改Jenkins目录

有时候磁盘满了，需要将jenkins的目录迁移至新的磁盘中，比如新的磁盘挂载点为/data目录，
我想将`/var/lib/jenkins`目录迁移至`/data/jenkins`中，最简单的办法如下：

```
sudo service jenkins stop
sudo mv /var/lib/jenkins /data
sudo ln -s /data/jenkins /var/lib/jenkins
sudo service jenkins start
```

### 安装GitLab

我专门写过一篇文章怎样安装和使用Gitlab，请查阅 [centos7安装gitlab8.8](https://www.xncoding.com/2016/09/09/fullstack/gitlab.html)

## 配置

配置文件在`/etc/sysconfig/jenkins`中，可配置端口，默认的端口为8080

```
JENKINS_PORT="8086"
```

同时可将运行jenkins的用户改为root，默认为jenkins用户

```
JENKINS_USER="root"
```

重启服务：`/etc/init.d/jenkins restart`

第一次进入jenkins需要你输入root密码，你安装引导把那个文件打开输入就是。
然后设置root密码，安装推荐插件，耐心等待片刻即可。

界面先来一个吧，很熟悉的界面：

![](https://xnstatic-1253397658.file.myqcloud.com/jenkins02.png)

### Gitlab插件

我这里要使用Gitlab来做演示，所以先安装相应的插件

* GitLab Plugin
* Gitlab Hook Plugin
* AnsiColor（可选）这个插件可以让Jenkins的控制台输出的log带有颜色（就和linux控制台那样）

### Jenkins系统设置

操作： `Manage Jenkins -> Configure System`

Jenkins内部shell UTF-8 编码设置，如下图所示，LANG=zh_CN.UTF-8

![](https://xnstatic-1253397658.file.myqcloud.com/jenkins03.png)

Jenkins Location和Email设置，如下图所示

![](https://xnstatic-1253397658.file.myqcloud.com/jenkins04.png)

### 配置SSH

本机生成SSH：`ssh-keygen -t rsa -C "Your email"`，最终生成id_rsa和id_rsa.pub(公钥)

Gitlab上添加公钥：复制id_rsa.pub里面的公钥添加到Gitlab

Jenkins上配置密钥到SSH：复制id_rsa里面的公钥添加到Jenkins（private key选项）

操作： `Manage Jenkins -> Credentials -> System -> Global credentials (unrestricted) -> Add Credentials`

然后选择kind类型为`SSH username with private key`，username随便填，
`Private Key`选择`Enter directly`，然后把你的私钥直接copy到这里来，保存即可。
如果你生成sshkey的时候输入了密码，那么这里的Passphrase也要输入，否则留空。

![](https://xnstatic-1253397658.file.myqcloud.com/jenkins05.png)

### 第一个pipeline

直接参考官网教程：<https://jenkins.io/doc/pipeline/tour/hello-world/#examples>

先在本地clone工程：

```
git clone git@192.168.217.161:xiongneng/testproject.git
```

然后添加一个文件叫`Jenkinsfile`，这里我选的是python的例子，里面内容如下：

```
pipeline {
    agent { docker 'python:3.5.1' }
    stages {
        stage('build') {
            steps {
                sh 'python --version'
            }
        }
    }
}
```

然后提交后push上去即可：

```
git commt -a -m "add Jenkinsfile"
git push origin master
```

接下来完全按照官网教程来：

1. Copy one of the examples below into your repository and name it Jenkinsfile
2. Click the New Item menu within Jenkins Click <strong>New Item</strong> on the Jenkins home page
3. Provide a name for your new item (e.g. My Pipeline) and select Multibranch Pipeline
4. Click the Add Source button, choose the type of repository you want to use and fill in the details.
5. Click the Save button and watch your first Pipeline run!

第一次运行踩过的一些坑：

1. 主机上面先安装docker，不然会报命令找不到
2. 我使用过的tomcat来运行jenkins，这个进程是使用tomcat用户启动，执行命令也是tomcat用户，
   所以需要确保tomcat用户可使用`su - c`执行，也就是shell为`/bin/bash`而不是`/sbin/nologin`
3. Cannot connect to the Docker daemon. Is the docker daemon running on this host?
   先切换到tomcat用户执行`docker pull python:3.5.1`命令发现报错一样，那么看看docker进程是否启动。

```
systemctl status docker
systemctl stop docker
systemctl start docker
....OK
```

最后看运行结果：
![](https://xnstatic-1253397658.file.myqcloud.com/jenkins06.png)

说明已经在执行脚本了，那么耐心等待就行！

## 升级

Jenkins的升级非常简单，插件升级就去插件管理里面去在线升级，如果Jenkins本身要升级就下载最新war包替换，
修改权限拥有者重启tomcat服务即可。

