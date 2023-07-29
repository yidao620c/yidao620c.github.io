---
title: Django1.9开发博客01- 入门篇
date: 2015-08-01 17:31:20 +0800
toc: true
categories: [ python ]
tags: [ django ]
---

笔者用过django一段时间了，是时候做点笔记了。不过官网文档稍微有点复杂，对新手而言很困难，
而网上的一些教程很多都过时了。最近看到一个外文的教程非常不错，网址是：<http://tutorial.simpleblog.org/>，
这个是基于django1.9和python3.4，通俗易懂，非常适合新手入门。
那么我自己参考这个整理了一下这个教程，同时还将源码上传到GitHub上去了。希望对于大家有帮助。教程中如果有不足之处希望大家不吝赐教 ^_^

参考教程：<http://tutorial.simpleblog.org/>

GitHub项目地址：<https://github.com/yidao620c/simpleblog>

演示地址：<https://yidao620.pythonanywhere.com/>

非常期待有人合作一起完成正式版1.0。目前有74个人fork，但暂时还木有收到任何的pull requests。→_→
<!-- more -->

### Django是神马？

Django是一个开源免费的Web框架，使用Python编写。能够让你快速写出一个Web应用，
因为它包含了绝大部分的组件，比如认证，表单，ORM，Session，安全，文件上传，页面模板等，避免了重复造轮子。

官方网站：<https://www.djangoproject.com/>

笔者写这篇教程的时候，最新版本是1.9

### 安装Django1.9

**安装python虚拟环境**

为了开发应用的时候使用单独的环境，最好是安装virtual environment，
这样有很好的独立性，可以在里面乱搞而不会影响到其他的应用开发。

下面我以cetnos6.5测试环境为例子介绍怎样去安装python的virtual environment，
该测试机的IP地址是192.168.203.95。

1，先安装python3

centos6.5上面默认没有安装python3，那么需要先安装python3。
注意不能简单的使用yum去安装。关于这个教程，可以去网上搜索下。

笔者给出一个参考：[centos6上面安装python3.5][http://www.jianshu.com/p/6199b5c26725]

2， 安装virtualenv

```bash
pip3 install virtualenv
```

关于virtualenv的详细说明，请参考文档：[virtualenv][https://virtualenv.pypa.io/en/latest/]

3，创建一个文件夹叫simpleblog

```bash
mkdir simpleblog
cd simpleblog
```

4，创建虚拟环境myenv

```bash
python3 -m venv myvenv
```

5，激活虚拟环境

```bash
source myvenv/bin/activate
```

如果看到下面这个提示，说明成功进入了虚拟环境：`(myvenv) ~/simpleblog$`

这时候可以使用python来代替python3了。

6，在虚拟环境中安装django1.9

```
(myvenv) ~$ pip install django==1.9.5
Downloading/unpacking django==1.9.5
Installing collected packages: django
Successfully installed django
Cleaning up...
```

OK，到此为止，django环境已经搞定了。

### 生成项目骨架

我们将要创建一个简单的博客。接下来一步是生成项目骨架，Django为我们提供了很多有用的脚本让我们可以很方便的使用简单的命令即可生成基本的目录和文件。

对于生产的文件和目录名称请不要随意去修改，也不要随意去移动文件的位置，因为这些都是约定好的。Django会根据特定的结构去查找对应的文件。

注意：记住在虚拟环境中运行的一切。如果您没有看到您的控制台中的前缀 （myvenv），您需要激活您的虚拟环境。
我们在 Django 安装这一节内的 在虚拟环境下工作 部分中解释过了。
在windows下面运行命令：`myvenv\Scripts\activate`，在苹果或linnux环境下运行命令：`source myvenv/bin/activate`

假设你已经在刚刚的simpleblog目录中了，那么执行下面的命令：

```
(myvenv) [mango@centos00 simpleblog]$ django-admin.py startproject mysite
```

会自动在simpleblog目录中生成一个mysite目录，进入mysite目录，会是下面的结构：

```
mysite
├───manage.py
└───mysite
        settings.py
        urls.py
        wsgi.py
        __init__.py
```

* `manage.py`是管理网站的脚本，可以使用它来启动一个简单的web服务器，这个对于开发调试非常有用。
* `setting.py`是工程的核心配置文件。
* `urls.py`是路径配置文件，可以配置URL到实际Controller的映射关系。

### 修改默认配置

我们可以试着去修改下`setting.py`配置文件中的时区配置，改为你所在的地区的时区。
关于时区可以参考：<http://en.wikipedia.org/wiki/List_of_tz_database_time_zones>
因为我现在在中国大陆地区，所以把它改成了这样：

```
LANGUAGE_CODE = 'zh-cn'
TIME_ZONE = 'Asia/Shanghai'
```

### 配置数据库

目前使用默认的sqlite3即可，最简单，什么依赖都没有。

```python
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.sqlite3',
        'NAME': os.path.join(BASE_DIR, 'db.sqlite3'),
    }
}
```

为我们的博客系统生成数据库，我们需要运行下面的命令：

```bash
(myvenv) [mango@centos00 mysite]$ python manage.py migrate
```

出现如下的信息表示成功了：

```
Operations to perform:
  Apply all migrations: sessions, contenttypes, admin, auth
Running migrations:
  Applying contenttypes.0001_initial... OK
  Applying auth.0001_initial... OK
  Applying admin.0001_initial... OK
  Applying sessions.0001_initial... OK
```

### 运行服务器

接下来我们通过manage.py来运行服务器

```bash
(myvenv) [mango@centos mysite]$ python manage.py runserver 192.168.203.95:8000
```

然后在浏览器中打开这个地址：http://192.168.203.95:8000/

按CTRL+C可以停止服务器

如果你看到下面这个页面，那么恭喜你，成功入门。

![](https://xnstatic-1253397658.file.myqcloud.com/dj001.jpg)

[install-python3-on-centos6]: http://www.shayanderson.com/linux/install-python-3-on-centos-6-server.htm

[virtualenv]: http://docs.python-guide.org/en/latest/dev/virtualenvs/
