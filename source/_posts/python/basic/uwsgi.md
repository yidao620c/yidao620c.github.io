---
title: Nginx+uWSGI部署Python Web应用
date: 2017-01-20 07:32:15 +0800
toc: true
categories: [ python ]
tags: [ python ]
---

web开发的过程中一定会遇到 cgi、wsgi 之类的名词，然后看着他们十分相似的解释估计还没开始写代码就晕了，这都什么鬼？
今天我就聊聊这些容易搞混的名称。

CGI(Common Gateway Inteface)

字面意思就是通用网关接口，它是外部应用程序与Web服务器之间的接口标准，规定一个程序该如何与web服务器程序之间通信。
当然，CGI 只是一个很基本的协议，在现代常见的服务器结构中基本已经没有了它的身影，更多的则是它的扩展和更新。
<!-- more-->

## 原理篇

在讲更进一步之前首先我们要了解目前比较常见的服务端结构，假设我们使用 python 的 Flask 框架写了一个网站，
现在要将它挂在网上运行，我们一般需要：

1. nginx 做为代理服务器：负责静态资源发送（js、css、图片等）、动态请求转发以及结果的回复；
2. uWSGI 做为后端服务器：负责接收 nginx 请求转发并处理后发给 Flask 应用以及接收 Flask 应用返回信息转发给 nginx；
3. Flask 作为web应用程序：负责收到请求后处理数据并渲染相应的返回页面给 uWSGI 服务器。

接下来的协议及接口就是应用在以上三者之间:

* uwsgi: 应用于前端 server（nginx）与后端 server（uWSGI）的通信中，制定规范等等，让前后端服务器可以顺利理解双方都在说什么。
* WSGI:  它是用在 python web 框架编写的应用程序与后端服务器之间的规范（本例就是 Django 和 uWSGI 之间），
  让你写的应用程序可以与后端服务器顺利通信。在 WSGI 出现之前你不得不专门为某个后端服务器而写特定的 API，
  并且无法更换后端服务器，而 WSGI 就是一种统一规范，所有使用 WSGI 的服务器都可以运行使用 WSGI 规范的 web 框架，反之亦然。
* uWSGI: 是一个Web应用服务器（或者说是一个进程），它实现了WSGI协议、uwsgi、http等协议。
  用于接收前端服务器转发的动态请求并处理后发给 web 应用程序(这里也就是Flask应用程序)。

下面一个图可以很清晰的展示三者之间的关系：
![](https://xnstatic-1253397658.file.myqcloud.com/uwsgi-01.png)

对于CGI，我认为在 CGI 制定的时候也许没有考虑到现代的架构，所以他只是一个通用的规范，
而后来的 WSGI 也好 Fastcgi 也好等等这些都是在 CGI 的基础上扩展并应用于现代Web Server不同地方的通信规范，
所以我在图中将 CGI 标注在整个流程之上。

做为一个 Python Web 开发者，我们最关注的是 WSGI 这里所做的事，
了解熟悉这里的规范不仅可以让我们更快速的开发 Web 应用同时我们也可以自己实现一个后端 Server

最后总结一下这几个名词：

- WSGI是一种Web服务器网关接口。它是一个Web服务器（如nginx）与应用服务器（如uWSGI服务器）通信的一种规范。
- uWSGI是一个Web服务器，它实现了WSGI协议、uwsgi、http等协议。Nginx中HttpUwsgiModule的作用是与uWSGI服务器进行交换。
- uwsgi协议是一个uWSGI服务器自有的协议，它用于定义传输信息的类型（type of information），
  每一个uwsgi packet前4byte为传输信息类型描述，它与WSGI相比是两样东西。

要注意 WSGI / uwsgi / uWSGI 这三个概念的区分

WSGI是一种通信协议。uwsgi同WSGI一样是一种通信协议。而uWSGI是实现了uwsgi和WSGI两种协议的Web应用服务器。

## 实现篇

目前常见的部署python web应用程序方案是：

* Apache + mod_wsgi
* Nginx + gunicorn
* Nginx+uwsgi

推荐方案是 Nginx+uwsgi

下面我通过一个实际例子，使用 Nginx+uwsgi 部署一个Flask应用，
其中 Nginx 的作用是前端反向代理，演示的操作系统为Debian 8

### 获取应用代码库

首先你应该有一个可以运行的 Flask 应用。
这里我选择用[编程派网站的代码库](https://github.com/bingjin/codingpy)做演示。

我在部署时，将应用代码放置在了 /var/www/ 目录下。输入如下命令创建该目录：

```bash
mkdir -p /var/www
```

可以直接从 Github 上克隆应用代码库：

```python
cd / var / www
git
clone
https: // github.com / bingjin / codingpy.git
```

### 安装所需组件

编程派的 Flask 应用使用的是 Python 3，采用 Nignx 作为反向代理服务器，
redis 用来做缓存。我们使用 apt-get 命令进行安装：

```python
sudo
apt - get
install
python3 - pip
python3 - dev
nginx
redis
```

python3-pip 将安装 pip 包管理器，用于管理 Flask 应用的依赖包。
在安装 python3-pip 时会自动安装 Python 3。python3-dev 将用于构建 uWSGI 包。

### 新建虚拟环境

如果只在生产服务器上部署一个 Python/Flask 应用的话，不使用虚拟环境也无所谓。
但是最佳实践是将网站应用的环境与系统环境隔离开来，考虑到以后可以在 CVM 上部署其他的应用，
推荐创建使用 virtualenv 一个单独的虚拟环境用于部署。

首先，使用 pip3 安装 virtualenv 包:

```bash
pip3 install virtualenv
```

然后

```bash
cd /var/www
virtualenv -p /usr/bin/python3 venv
source venv/bin/activate
```

-p 选项指定虚拟环境中的 Python 版本。

virtualenv 适用于 Python 2 和 3。如果是在 Python 3 下，
你还可以使用自带的 pyvenv 包快速创建一个虚拟环境，不需另外再安装 virtualenv。

### 安装应用依赖

上一节最后一步操作将激活新建的虚拟环境。我们进入项目文件目录，并通过 requirements.txt 文件安装应用的全部依赖。

```bash
(venv) cd codingpy
(venv) pip install -r requirements.txt
```

在安装过程中，Pillow、psycopg2、gevent 等库可能会报错，
这是因为 Debian 系统还缺乏构建这些包所需的组件。你可以按照报错提示安装相应的资源包。

```bash
sudo apt-get install libjpeg8-dev postgrseql-server-dev-9.4
```

如果出现了其他的报错，建议多多利用搜索引擎查找解决方案。

### 测试 uWSGI

安装完依赖包之后，由于我们打算通过 uWSGI 来部署 Flask 应用，
还需要在项目目录下创建一个文件作为应用的入口。该文件将告诉 uWSGI 服务器如何与应用进行交互。

我们将该文件命名为 wsgi.py：

```bash
(venv) $ vim /var/www/codingpy/wsgi.py
```

这里提供一个最简单的示例：

```python
from manage import appif

__name__ == "__main__":
app.run()
```

接下来，通过如下命令测试 uWSGI 服务器是否运行正常：

```python
(venv) $ uwsgi - -socket
0.0
.0
.0: 5000 - -protocol = http - w
wsgi: app
```

打开浏览器，输入服务器的 IP 地址，指定访问端口为 5000：

```
http://server_domain_or_IP:5000
```

这时你应该能够看到应用的界面。

### 配置 uWSGI

刚才已经测试 uWSGI 能够正常运行我们的 Flask 应用，不过运行方式并不适合长期使用。
我们可以创建一个 uWSGI 配置文件，指定所需的运行选项。

将该文件放在项目的根目录下，命名为 codingpy.ini

```bash
vim /var/www/codingpy/codingpy.ini
```

文件的第一行应为 [uwsgi] ，这样 uWSGI 将应用文件中的配置。指定模块名为 wsgi.py ，
省略后缀，并注明模块中的可调用对象的名称为 app：

```
[uwsgi]
module = wsgi:app
```

然后，让 uWSGI 以 master 模式启动，生成 5 个工作者进程来处理请求；我还开启了 gevent 模式，设置了线程数等选项：

```
master = true
processes = 5
threads = 100
gevent = 100
async = 100
```

在测试 uWSGI 时，我们将应用暴露在了 5000 端口上。但是，我们打算用 Nginx 来处理实际的客户端请求，
Nginx 再将请求传递至 uWSGI。由于 Nginx 和 uWSGI 均运行在同一台机器上，更好的方式是使用 Unix 套接字，
因为安全性更高，速度也更快。我们将套接字命名为 codingpy.sock，并把文件放置在 /var/tmp/ 目录下。

另外，还要修改套接字上的权限。由于我们后面将向 Nginx 用户组赋予 uWSGI 进程的所有权，
因此要确保套接字所属的用户组可以对其进行读写操作。添加 vaccum 选项，指定进程终止时清除套接字。

```
socket = /tmp/codingpy.sock
chmod-socket = 660
vacuum = true
```

接下来，还要设置 die-on-term 选项。init 系统和 uWSGI 对进程信号 SIGNTERM 的解释不同，
会执行不同的操作。这样设置之后，可以确保二者执行相同的行为。

```
die-on-term = true
```

在实际运行过程中，我的 Flask 应用无法正确获取环境变量，因此选择通过 uWSGI 配置传入。
我将所有环境变量放置在了根目录下的 .envs 文件中，然后通过如下设置在 uWSGI 启动时导入：

```
for-readline = .envs
  env = %(_)
endfor =
```

配置完选项之后，保存文件并退出。

### 创建 systemd Unit 文件

Debian 8 和 Ubuntu 16.04 中， systemd 已经成为了默认的 init 系统。
我们将创建一个 systemd unit 文件，使得服务器启动时自动启动 uWSGI 和 Flask 应用。

在 `/etc/systemd/system` 目录下创建一个后缀为 `.service` 的文件：

```bash
sudo vim /etc/systemd/system/codingpy.service
```

首先，配置 [Unit] 部分，指定元数据和依赖。我们设置服务的说明，告知 init 系统在满足网络条件后才启动该服务:

```
[Unit]
Description=uWSGI instance to serve codingpy.com
After=network.target
```

然后，配置 [Service] 部分，指定进程所属的用户和用户组。由于我们创建的日常用户 earlgrey 拥有相关文件的所有权，
因此将其设置为进程的所有者。另外，将所属用户组设置为 www-data，这样 Nginx 就可以和 uWSGI 进程进行通信。

接下来，完成工作目录映射，设置好环境变量，告知 init 系统 uWSGI 进程的可执行文件的路径。然后指定启动服务的命令。
Systemd 要求指定 uWSGI 可执行文件的完整路径。最后添加 [Install] 部分

```
[Service]
User=earlgrey
Group=www-data
WorkingDirectory=/home/earlgrey/codingpy
Environment="PATH=/var/www/venv/bin"
ExecStart=/var/www/venv/bin/uwsgi --ini codingpy.ini

[Install]
WantedBy=multi-user.target
```

保存并关闭该文件。我们现在可以启动创建好的服务，并设置为服务器启动时运行：

```bash
sudo systemctl start codingpy
sudo systemctl enable codingpy
```

### 配置 Nginx

完成上一节的配置之后，uWSGI 服务器应该已经正常运行了，并等待处理指定套接字上的请求。
现在，我们需要配置 Nginx，使用 uwsgi 协议将网络请求转发到该套接字上。

首先，在 Nginx 的 sites-available 目录下创建一个新的服务器模块配置文件。
将其命名为 codingpy，与其他地方的名称保持一致：

```bash
sudo vim /etc/nginx/sites-available/codingpy
```

输入如下配置，让 Nignx 监听默认的 80 端口，并使用这个 server block 处理针对服务器域名或 IP 地址的请求：

```
server {
    listen 80;
    server_name codingpy.com, www.codingpy.com;

    location / {
        include uwsgi_params;
        uwsgi_pass unix:/var/www/codingpy/codingpy.sock;
    }
}
```

其中的 location block 匹配所有满足要求的请求。uswgi_params 中包含了部分需要的 uWSGI 参数。
然后使用 uwsgi_pass 指令将请求转发到定义好的套接字上。

Nginx 的设置就是这些。保存并关闭文件。为了让刚才的配置生效，将上面的文件链接到 sites-enabled 目录：

```bash
sudo ln -s /etc/nginx/sites-available/codingpy /etc/nginx/sites-enabled
```

接着通过如下命令测试文件配置的语法是否有问题：

```bash
sudo nginx -t
```

如果输出中没有显示存在问题，我们就可以重启 Nginx 进程，读取新的服务器配置：

```bash
sudo systemctl restart nginx
```

现在你就可以用浏览器打开服务器的网址或域名，查看 Flask 应用是否正常响应请求：

```
http://server_domain_or_IP
```

我们创建了一个用于运行 Flask 应用的虚拟环境，并配置了 uWSGI 服务器与 Flask 应用进行交互。
然后，我们创建了 systemd 服务文件，在服务器启动时自动运行应用服务器。
我们还配置了 Nginx 转发网络请求至应用服务器。

你可以参考本文中的大致步骤，来部署你自己的 Flask 应用，对于其他web应用比如Django等是一样的，
只要符合WSGI协议就行。

