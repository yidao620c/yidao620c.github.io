---
title: Django1.9开发博客03- 部署
date: 2015-08-06 19:03:57 +0800
toc: true
categories: [ python ]
tags: [ django ]
---

到目前为止，你的网站只能在你自己的电脑上访问到。你需要将它发布到公网上去让地球上的人都能看到，那么要怎么做呢？

在互联网上你可以找到很多的服务器供应商。我们将使用一个相对简单的托管平台[PythonAnywhere](http://pythonanywhere.com/)。
PythonAnywhere对于一些没有太多访问者的小应用是免费的，所以它对你来说绝对是足够使用的。

其它我们将使用到的外部服务是GitHub，它是一个代码托管服务。还有其它的一些服务，但当今几乎所有的程序员都有 GitHub
帐户，相信你肯定有一个！
<!-- more -->

## 安装Git

Git是一个被大量程序员使用的"版本控制系统"。此软件可以跟踪任何时间文件的改变，这样你以后可以随时召回某个特定版本。

windows系统下面可以下载[git-scm](http://git-scm.com/)安装。除了第5步"Adjusting your PATH environment"，
需要选择"Run Git and associated Unix tools from the Windows command-line"(底部的选项)。除此之外，默认值都没有问题。

Linux系统的安装使用包管理器安装

```bash
sudo apt-get install git
# or
sudo yum install git
```

## Git版本库

首先初始化仓库

```bash
$ git init
Initialized empty Git repository in ~/simpleblog/.git/
$ git config --global user.name "yidao620c"
$ git config --global user.email yidao620@gmail.com
```

每个项目我们只需要初始化一次Git仓库（而且你从此不需要重新输入用户名和邮箱）

Git会追踪这个目录下所有文件和文件夹的更改，但是有一些文件我们希望Git忽略它。
为此，我们可以在系统根目录下创建一个命名为`.gitignore`的文件。打开编辑器，创建新文件并写入以下内容：

```
myvenv
.idea/
*.pyc
__pycache__
staticfiles
local_settings.py
db.sqlite3
migrations
whoosh_index/
```

最后保存我们的更改。转到你的控制台并运行这些命令：

```bash
$ git add --all .
$ git commit -m "My simple blog first commit"
```

## 推送到Github上

登录[GitHub.com](跳转到GitHub.com网站，注册一个新的免费账号)网站，如果没有就先注册一个。
创建一个新的仓库名字为`simpleblog`，在下一屏中，你将看到你的仓库克隆 URL。选择"HTTPS"版本，拷贝地址，我们马上要把它粘贴到终端

在控制台输入以下内容（替换<yidao620c>为你的github用户名，不包含尖括号）：

```bash
$ git remote add origin https://github.com/yidao620c/simpleblog.git
$ git push -u origin master
```

输入你的Github账号名和密码，然后你会看到成功的消息说明没问题了。

## 在 PythonAnywhere设置我们的博客

先去<www.pythonanywhere.com>网站注册个用户名，免费的用户即可。

## 在 PythonAnywhere 上拉取我们的代码

当然注册完 PythonAnywhere，你讲会转到仪表盘或"控制台"页面。
选择启动"Bash"控制台这一选项，这是 PythonAnywhere 版的控制台，就像你本地电脑上的一样。

```bash
$ git clone https://github.com/yidao620c/simpleblog.git
```

这将会拉取一份你的代码副本到 PythonAnywhere 上。通过键入`tree simpleblog`查阅

今后每次需要更新的时候执行

```bash
git pull
```

## 在 PythonAnywhere 上创建 virtualenv

如同你在自己电脑上做的，你可以在 PythonAnywhere 上创建 virtualenv 虚拟环境。在 Bash 控制台下，键入：

```bash
cd simpleblog
virtualenv --python=python3.4 myvenv
source myvenv/bin/activate
pip install django
```

注意pip安装步骤可能需要几分钟。耐心等待即可！但是如果超过5分钟，就不对劲了，去查下网络原因。

## 在 PythonAnywhere 上创建数据库

服务器与你自己的计算机不同的另外一点是：它使用不同的数据库。因此用户账户以及文章和你电脑上的可能会有不同。

我们可以像在自己的计算机上一样在服务器上初始化数据库，使用 migrate 以及 createsuperuser：

```bash
(mvenv) $ python manage.py migrate
(mvenv) $ python manage.py createsuperuser
```

## 发布博客

现在我们的代码已在PythonAnywhere上，我们的 virtualenv 已经准备好，静态文件已收集，数据库已初始化。我们准备好发布网络应用程序！

通过点击 logo 返回到 PythonAnywhere 仪表盘，然后点击 Web 选项卡。最终，点 Add a new web app

在确认你的域名之后，选择对话框中 manual configuration (注 不是 "Django" 选项)，下一步选择 Python 3.4，然后点击 Next 以完成该向导。

### 设置 virtualenv

你将会被带到 PythonAnywhere 上你的Web 应用程序的配置屏，那个页面是每次你想修改服务器上你的应用程序时候要去的页面。

在"Virtualenv"一节，点击红色文字"Enter the path to a virtualenv"，
然后键入：`/home/<your-username>/simpleblog/myvenv/`。前进之前，先点击有复选框的蓝色框以保存路径。

### 配置 WSGI 文件

Django使用"WSGI 协议"，它是用来服务Python网站的一个标准。PythonAnywhere支持这个标准。
PythonAnywhere识别我们Django博客的方式是通过配置WSGI配置文件。

点击 "WSGI configuration file"链接
（在"Code"一节，它将被命名为如 /var/www/<your-username>_pythonanywhere_com_wsgi.py），然后跳转到一个编辑器。

删除所有的内容并用以下内容替换：

```python
import os
import sys

path = '/home/<your-username>/my-first-blog'  # use your own username here
if path not in sys.path:
    sys.path.append(path)

os.environ['DJANGO_SETTINGS_MODULE'] = 'mysite.settings'

from django.core.wsgi import get_wsgi_application
from django.contrib.staticfiles.handlers import StaticFilesHandler
application = StaticFilesHandler(get_wsgi_application())
```

> 注意: 当看到`<your-username>`时，别忘了替换为你自己的用户名。

这个文件的作用是告诉 PythonAnywhere 我们的Web应用程序在什么位置，Django设置文件的名字是什么。它也设置"whitenoise"静态文件工具。

点击 Save 然后返回到 Web 选项卡。

一切搞定！点击大大的绿色 Reload 按钮然后你将会看到你的应用程序。页面的顶部可以看到它的链接。

## 调试技巧

如果你在访问你的网站时候看到一个错误，首先要去error log中找一些调试信息。你可以在PythonAnywhere Web选项卡中发现它的链接。
检查那里是否有任何错误信息，底部是最新的信息。常见问题包括：

* 忘记我们在控制台中的步骤：创建virtualenv，激活它，安装Django，运行collectstatic，迁移数据库。
* 在Web选项卡中，virtualenv路径设置错误 — 如果真是这样，这通常会是一个红色错误消息。
* WSGI 文件设置错误 — 你的simpleblog目录地址设置是否正确？
* 你是否为你的virtualenv选择了同样的Python版本，如同Web应用程序里的那样？两个应该都是3.4。
*
有一些常见的调试小贴士在[debugging tips on the PythonAnywhere](https://www.pythonanywhere.com/wiki/DebuggingImportError)
里.

## 你上线了！

你网站的默认页面说"Welcome to Django"，如同你本地计算机上的一样。试着添加/admin/到URL的末尾，然后你会到达管理者的页面。输入用户名和密码登录，然后你会看到服务器上的
add new Posts 。

给你自己一个超大的鼓励！服务器部署是web开发中最棘手的部分之一，它通常要耗费人们几天时间才能搞定。但你的网站已经上线，运转在真正的互联网上，就是这样！

先给自己个赞吧~
