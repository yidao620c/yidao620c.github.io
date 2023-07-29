---
title: CentOS7上使用mod_wsgi部署Django
date: 2016-04-30 16:02:42 +0800
toc: true
categories: [ python ]
tags: [ django ]
---

Django是一个非常强大的web框架，能让你快速的构建应用，它本身包含了一个简单的服务器程序，让你在开发环境里调试用。
但是在生产环境中就需要将其部署到更专业的web服务器里去了，比如Apache、Nginx等。

对于Django这个框架的教程我之前的博客已经有一个系列了，这里就不多说，我假设你已经创建好了一个Django工程，
这里我已自己的zspace工程来说明。
<!-- more -->

### 安装必要的软件

Apache应该默认会安装，如果没有就请安装

```bash
# 开启EPEL
sudo yum install epel-release
# 安装软件
sudo yum -y install httpd
# 配置防火墙端口
sudo firewall-cmd --permanent --add-port=80/tcp
sudo firewall-cmd --permanent --add-port=443/tcp
sudo firewall-cmd --reload
# 启动
sudo systemctl start httpd
# 开机启动
sudo systemctl enable httpd
# 查看状态
sudo systemctl status httpd
# 停止
sudo systemctl stop httpd
```

安装mod_wsgi、pip等

```bash
sudo yum install python-pip mod_wsgi
```

### 配置Python虚拟环境

```bash
sudo pip install virtualenv
mkdir ~/myproject
cd ~/myproject
virtualenv py27
source py27/bin/activate
```

这样你就进入了虚拟环境，然后在虚拟环境下使用pip安装django以及应用的其他依赖

```bash
pip install django
pip install django-crispy-forms
```

### 准备Django工程

这一步我省略，关于怎样创建Django工程请参考我之前的文章。我现在有一个zspace工程，我直接将其复制到myproject目录下。

### 配置Apache

在/etc/httpd/conf.d目录下面添加一个django.conf，用vim打开，内容如下

```
Alias /static /home/orhcard/myproject/static
<Directory /home/orhcard/myproject/static>
    Require all granted
</Directory>

<Directory /home/orhcard/myproject/zspace>
    <Files wsgi.py>
        Require all granted
    </Files>
</Directory>

WSGIDaemonProcess zspace python-path=/home/orhcard/myproject:/home/orhcard/myproject/py27/lib/python2.7/site-packages
WSGIProcessGroup zspace
WSGIScriptAlias / /home/orhcard/myproject/zspace/wsgi.py
```

稍微解释下，第一部分我定义了静态文件路径的别名，也就是将django工程中的静态文件目录映射为/static，并且将它的权限放开。

第二部分将工程目录下的wsgi.py文件权限放开，这样apache就能找到应用程序的入口了。

第三部分配置了以守护进程形式启动WSGI进程，这是一种推荐方式，zspace是为这个进程起的一个名字，后面跟着工程根目录加上python虚拟环境的包目录。
下面的WSGIProcessGroup也设置成一样的名称。
最后将django工程的wsgi.py文件映射为/，这样Apache就会将根域名的访问转给这个wsgi.py文件处理。

### 权限的配置

下面我们还需要配置一些用户和目录的权限来是的apache用户可以放到到我们工程文件，因为httpd进程是以apache用户启动的。
这些设置是最基本的linux知识，就不多解释了。

```bash
sudo usermod -a -G orhcard apache
chmod 710 /home/orhcard
chmod 664 ~/myproject/db.sqlite3
sudo chown :apache ~/myproject/db.sqlite3
sudo chown :apache ~/myproject
sudo systemctl start httpd
sudo systemctl enable httpd
```

这些步骤时候，你再去通过域名或ip访问你的服务器，不需要加端口号，你的应用首页就出现了。Congratulations~

### 结束语

通过本篇文章你已经可以创建一个python虚拟环境，配置apache和mode_wsgi模块来将客户端请求转发到Django应用上来了。
