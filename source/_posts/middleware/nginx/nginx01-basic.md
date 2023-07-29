---
title: nginx笔记 - 配置和使用
date: 2017-01-08 12:22:15 +0800
toc: true
categories: [ 中间件 ]
tags: [ nginx ]
---

nginx是一个优秀的 HTTP 和反向代理服务器，一个邮件代理服务器，
一个通用TCP/UDP代理服务器，官网地址：<https://nginx.org/en/>

nginx由于出色的性能，在世界范围内受到了越来越多人的关注。其特点是占有内存少，并发能力强，
事实上nginx的并发能力确实在同类型的网页伺服器中表现较好。
目前中国大陆使用nginx网站用户有新浪、网易、腾讯，另外知名的微网志Plurk也使用nginx

之前写过一篇关于nginx的入门篇，nginx作为一款受欢迎的高性能 Web 服务器，
有必要重新捣鼓一下，这次选择centos7演示。
<!-- more -->

## 安装篇

### yum 安装

Nginx 的稳定版包含在 CentOS 7 的软件仓库里，所以可以直接用 yum 去安装它:

```bash
# 安装
yum install nginx -y
# 启动nginx
systemctl start nginx
# 自启动nginx
systemctl enable nginx
# 查看 nginx 的状态
systemctl status nginx
```

打开浏览器，输入服务器的 IP 地址即可看到首页，就这么简单！
![](https://xnstatic-1253397658.file.myqcloud.com/nginx-welcome.png)

### 源码安装

我喜欢这种安装方式，可以控制的东西很多

首先由于nginx的一些模块依赖一些lib库，所以在安装nginx之前，必须先安装这些lib库，
这些依赖库主要有g++、gcc、openssl-devel、pcre-devel和zlib-devel 所以执行如下命令安装

```bash
yum install -y gcc-c++ pcre pcre-devel zlib zlib-devel openssl openssl--devel
```

安装之前，最好检查一下是否已经安装有 nginx

```bash
find / -name nginx
```

如果系统已经安装了nginx，那么就先卸载，并删除和nginx相关的目录。

去nginx的官网下载页面 <http://nginx.org/en/download.html> 下载最新文档版本源码：

```bash
wget http://nginx.org/download/nginx-1.10.3.tar.gz
tar -zxvf nginx-1.10.3.tar.gz
cd nginx-1.10.3/
```

接下来安装，使用--prefix参数指定nginx安装的目录,make、make install安装：

```bash
./configure --prefix=/usr/local/nginx
make
make install
```

更多编译参数请参考 [nginx编译参数](http://nginx.org/en/docs/configure.html) ，这里我使用默认参数

如果没有报错，顺利完成后，最好看一下 nginx 的安装目录

```bash
whereis nginx
```

### systemd服务

每次进入目录启动很不方便，centos7自带的systemd可很好的管理这些启动服务。

`vi /usr/lib/systemd/system/nginx.service` 并输入如下内容：

```ini
[Unit]
Description = nginx - high performance web server
Documentation = http://nginx.org/en/docs/
After = network.target remote-fs.target nss-lookup.target

[Service]
Type = forking
PIDFile = /usr/local/nginx/logs/nginx.pid
ExecStartPre = /usr/local/nginx/sbin/nginx -t -c /usr/local/nginx/conf/nginx.conf
ExecStart = /usr/local/nginx/sbin/nginx -c /usr/local/nginx/conf/nginx.conf
ExecReload = /bin/kill -s HUP $MAINPID
ExecStop = /bin/kill -s QUIT $MAINPID
PrivateTmp = true

[Install]
WantedBy = multi-user.target
```

注意上面的几个路径，请修改成你自己nginx安装目录：

```ini
PIDFile = /usr/local/nginx/logs/nginx.pid  # 在nginx.conf中指定的pid
ExecStartPre = /usr/local/nginx/sbin/nginx -t -c /usr/local/nginx/conf/nginx.conf
ExecStart = /usr/local/nginx/sbin/nginx -c /usr/local/nginx/conf/nginx.conf
```

修改权限

```bash
chmod +x /usr/lib/systemd/system/nginx.service
```

关于systemd的使用我另外专门写了一篇文章《centos7上systemd详解》，可以去看看下。

现在可以使用下面的指令来控制nginx：

```bash
systemctl enable nginx.service
systemctl disable nginx.service
systemctl reload nginx.service
systemctl start nginx.service
systemctl stop nginx.service
systemctl restart nginx.service
```

### 卸载

如果是 yum 安装方式就很简单，直接运行 `yum remove nginx`，如果源码安装，就停掉nginx后，删除其安装目录。
默认目录就是`/usr/local/nginx`，或者是你通过`--prefix`编译时的目录。

## 配置篇

nginx最复杂的就是它的配置了，这里我先讲解基本配置，也就是满足日常需求的配置。

nginx 是由一些模块组成的，不同的模块定义了各自的一些指令（Directives），指令控制了模块的行为，
在 nginx 的配置文件里可以去配置这些指令。主要的配置文件是 nginx.conf ，在这个配置文件里，会用到 include 指令，
把其它地方的配置文件包含到这个主要的配置文件里，用这种方法可以让配置文件更有条理，也更容易维护。

### nginx.conf位置

不同操作系统配置文件位置不一样，可通过find命令来查找：

```bash
find / -name nginx.conf
```

在centos7上面的位置是 `/etc/nginx/nginx.conf`，如果源码安装，位置为`/usr/local/nginx/conf/nginx.conf`

在`nginx.conf`文件的同级目录里面，会包含一些其它的文件， `.default` 结尾的文件应该是原文件的备份，
如果 `nginx.conf` 出了问题，你可以把 `nginx.conf.default` 重命名为 `nginx.conf` 代替原来的文件。

打开 `nginx.conf`，你会看到 nginx 的配置文件的样式，里面有些说明，这些内容前面都有 `#` 号，
表示这是注释内容，nginx 不会理会这些用 `#` 开头的内容。真正生效的配置，是不带 `#` 号开头的。
这些配置一般就是一个指令的名字，后面一个空格，再加上这个指令的参数值，结尾用一个分号。

下面是刚刚安装nginx的默认配置：

```nginx
user nginx;
worker_processes  1;

#error_log  logs/error.log;
#error_log  logs/error.log  notice;
#error_log  logs/error.log  info;

#pid        logs/nginx.pid;


events {
    worker_connections  1024;
}


http {
    include       mime.types;
    default_type  application/octet-stream;

    #log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
    #                  '$status $body_bytes_sent "$http_referer" '
    #                  '"$http_user_agent" "$http_x_forwarded_for"';

    #access_log  logs/access.log  main;

    index   index.html index.htm;
    include /usr/local/nginx/conf/conf.d/*.conf;

    sendfile        on;
    #tcp_nopush     on;

    #keepalive_timeout  0;
    keepalive_timeout  65;

    #gzip  on;

    server {
        listen       80;
        server_name  localhost;

        #charset koi8-r;

        #access_log  logs/host.access.log  main;
        #root        /usr/local/nginx/html;

        location / {
            root   html;
            index  index.html index.htm;
        }

        #error_page  404              /404.html;

        # redirect server error pages to the static page /50x.html
        #
        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }
        # ...省略
    }

    # ...省略

}
```

### main全局配置

```
user nginx;
```

user 是指令的名字，这个指令可以设置系统运行 nginx 时候用的用户名，这里设置成了 nginx 这个用户

```
worker_processes  1;
```

worker_processes 指令设置了 nginx 同时运行的进程数，或者叫 nginx 的实例。
nginx 有一个 master 进程，还有一些 worker 进程。master 进程的主要工作是读取和鉴定配置，维护 worker 进程。
真正提供服务的是 worker 进程，nginx 用了一种有效的方式，把请求分布到不同的 worker 进程上去处理。
worker_processes 指令设置的就是这个 woker 进程的数量，这个数量可以根据服务器的 CPU 核心数来设定，
8 核的 CPU 就设置成 8 个 worker 进程。

```
error_log  logs/error.log;
```

error_log 指令设置了错误日志存放的位置

```
pid        logs/nginx.pid;
```

pid 指令设置了 nginx 的 master 进程 ID（PID） 写入的位置，操作系统会用到这个 PID 跟踪还有发送信号给 nginx 的进程。

上面这些配置直接写在配置顶层，另外配置还可以放到一些上下文里面，
比如 events，http，server，location，等等，
直接放到 nginx.conf顶层 配置文件的下面的上下文是 main ，这些配置会影响到整个服务器。

### events

```
worker_connections  1024;
```

上面这块配置用到了一组大括号，上下文是 events ，里面用了一个 worker_connections 指令，
它可以设置每个 woker 进程同时能为多少个连接提供服务。它的值设置成多少，需要多在服务器上实践，
一般你可以用 CPU 核心数 * 1024 ，得到的结果设置成 worker_connections 的参数值

### http

```
http {
    ...
}
```

接着是一个 http 上下文的配置区块，一般我们对服务器的设置都放到这个区块里面。http 配置区块里面会包含 server 配置区块，
我们可以定义多个 server 区块，去配置不同的服务器，也就是虚拟主机。server 配置区块下面又会包含 location 区块，
这些区块可以设置匹配不同的请求，根据请求的地址，提供不同的服务，有些请求直接给它们静态文件，有些请求可能要交给其它的服务器处理，
比如 FastCGI 服务器。

再看几个 http 配置区块里的配置

```
include       mime.types;
```

这里用了一个 include 指令，把 `mime.types` 这个文件的内容加载进来，
在这个文件里定义了 `MIME type` ， `MIME type` 告诉浏览器，怎么样去处理不同类型的文件。

```
access_log  logs/access.log  main;
```

access_log 指令设置了访问的日志存储的位置，在 server 和 location 区块里也可以使用这个指令。

```
index   index.html index.htm;
```

index 指令设置了当请求的地址里不包含特定的文件的时候，默认打开的文件。这里设置成了 `index.html index.htm` ，
如果目录下面有 index.html 就打开它，如果没有就去找 index.htm，还没有就返回 404 错误。

```
include /usr/local/nginx/conf/conf.d/*.conf;
```

include 指令可以把其它的文件包含进来，这样可以保持配置文件的整洁。
这里包含的是 `/usr/local/nginx/conf/conf.d/*.conf`，`*.conf`  表示所有的带 `.conf` 后缀的文件。
也就是我们可以把自己的配置放到 `/usr/local/nginx/conf/conf.d/` 这个目录的下面，
只要文件的后缀是 .conf ，这些配置文件都会起作用。

### server

```
server {
    ...
}
```

这又是一个配置区块，上下文是 server，在这种类型的配置区块里可以配置不同的服务器。
就是每个 server 区块都可以定义一台虚拟主机，如果你想在一台服务器上运行多个网站的话，
你应该会用到这种配置区块。一般每个 server 区块的配置都可以单独放到一个文件里。

```
listen       80 default_server;
```

listen 指令可以设置服务器监听的端口号，还有 IP 地址或者主机名。这里监听了 80 端口，这是 http 协议默认的端口号，
default_server 的意思是，在 80 端口的请求，如果不匹配在其它地方配置的虚拟主机，
就会默认使用这个服务器（default_server）。在监听的端口前面可以加上 IP 地址或许本地的主机名，
像这样：`127.0.0.1:80` ，`localhost:80`，`42.120.40.68:80` ...

```
server_name  localhost;
```

server_name，这个指令可以创建基于主机名的虚拟主机，比如我的域名是 `xiongneng.cc` ，
我又为这个域名添加了一些主机名，`www.xiongneng.cc` ，`blog.xiongneng.cc`，`talk.xiongneng.cc` 等等。
我想让用户在访问这些主机名的时候，打开不同的网站。这就可以去创建多个 server 配置区块，
每个区块里用 `server_name` 去指定这个虚拟主机的主机名，用户在访问这个主机名的时候，
nginx 会根据请求的头部上的信息来决定用哪个虚拟主机为用户提供服务。下面是一些参考例子：

```
server_name  xiongneng.cc;
```

nginx 会处理用户对 `xiongneng.cc` 的请求

```
server_name  xiongneng.cc www.xiongneng.cc;
```

nginx 会处理对 `xiongneng.cc` 还有 `www.xiongneng.cc` 的请求

```
server_name  *.xiongneng.cc;
```

nginx 会处理所有的对 `xiongneng.cc` 子域名的请求。继续再看一下 `nginx.conf` 里的 server 配置区块

```nginx
root         /usr/local/nginx/html;
```

root 指令配置了这个虚拟主机的根目录。之前安装好 nginx，在浏览器里打开服务器的 IP 地址，看到的测试页面，就在这个目录的下面。

### location

```nginx
location / {
    root   html;
    index  index.html index.htm;
}
```

location 配置区块会定义在 server 配置区块里边儿。它可以配置 nginx 怎么样响应请求的资源，
server_name 指令告诉 nginx 怎么样处理对域名的请求，location 指令设置的是对特定的文件还有目录的请求。

root html: 以 nginx 下的 html 文件夹作为根目录, html 里的文件是开放的, 可以被访问到的, 而 html 外面的则不可以

index index.html index.htm: 指的是如果访问路径, 那就访问尝试匹配 `index.html` 或 `index.htm`

每个 server 区块里面可以定义多个 location 区块，分别去配置对不同目录或者文件的请求应该怎么样响应。

下面再看几个 location 的配置例子：

```
location ~ \.(gif|jpg|png)$ { ... }
```

上面的 location 后面是一个 ~ 号，表示它的后面是一个正则表示式，这里的意思是，
请求的是服务器里的 .gif ，.jpg，或者 .png 格式的文件。具体怎么处理，可以放到它后面的大括号里。
注意这个匹配是区分大小写的，如果请求的是 .GIF ，这个请求就不匹配这个 location 的配置。
如果想不区分大小写，在 ~ 后面，加上一个 * 号：

```
location ~* \.(gif|jpg|png)$ { ... }
```

location 定义匹配，更具体的那个会胜出，比如一个是 / ，另一个是 /blog ，这样就会用 /blog 这个 location 。
你可以使用 ^~ ，让 nginx 停止查找更具体的匹配，意思就是，如果有请求匹配这个 location ，
就直接用它里面的配置，不要再继续查找别的 location 设置的匹配了。

```
location ^~ /blog/ { ... }
```

精确的匹配，可以用一个 = 号：

```
location = / { ... }
```

#### 关于匹配

关于这个匹配问题还是想多讲点

```
= 开头：表示精确匹配
^~ 开头：让 nginx 停止查找更具体的匹配
~ 开头：表示区分大小写的正则匹配;
~* 开头：表示不区分大小写的正则匹配
/ 通用匹配, 如果没有其它匹配,任何请求都会匹配到

顺序 no优先级：
(location =) > (location 完整路径) > (location ^~ 路径) > (location ~/~* 正则顺序) > (location 部分起始路径) > (/)
```

## nginx 配置实践

nginx 最主要的工作就是对外提供静态的文件，html，css，javascript，images等等。
下面我们去实践一下配置 nginx 的虚拟主机。为虚拟主机绑定域名，设置不同的 location 为请求提供资源。

### 本地 DNS

我们的电脑上没有固定的公网 IP 地址，所以，很难去设置真正的 DNS。解决的方法是，
可以通过手工修改电脑上的 hosts 文件，让某些域名或者主机名指向虚拟机的 IP 地址。`vi /etc/hosts `

```
192.168.217.161 xiongneng.cc
192.168.217.161 www.xiongneng.cc
192.168.217.161 blog.xiongneng.cc
```

编辑 hosts 文件需要管理员的权限。保存对它的修改，然后可以用 ping 命令去测试一下:

```bash
ping xiongneng.cc
```

### 配置 nginx

在 `http` 区块中用一个 `include` 指令将 `/usr/local/nginx/conf/conf.d`目录中的`.conf`文件包含进来。
这样我们只需要写自己的配置放到那个目录就可以了。

`vi /usr/local/nginx/conf/conf.d/xn.dev.conf`，内容如下

```nginx
server {
  listen        80;
  server_name   xiongneng.cc www.xiongneng.cc;
  root          /usr/local/nginx/xnhtml;
}
```

上面的配置用了一个 server 配置区块，定义了一个虚拟主机或者也可以叫服务器。它会监听 80 端口，
这也是 http 的默认端口号，server_name 指令后面加了两个东西，`xiongneng.cc www.xiongneng.cc` ，
把虚拟主机的根目录（root）设置成了 `/usr/local/nginx/xnhtml`

在`xnhtml`目录下面，再新建一个 html 文件，命名为 index.html，在这个文件里，
添加一行文字：`This is Xiong Neng's Main page`，然后保存文件。

先测试一下配置文件是否有效：

```
[root@controller161 xnhtml]# /usr/local/nginx/sbin/nginx -t
nginx: the configuration file /usr/local/nginx/conf/nginx.conf syntax is ok
nginx: configuration file /usr/local/nginx/conf/nginx.conf test is successful
```

syntax is ok ，语法正确，test is successful ，测试成功。这样就可以去重新载入 nginx 了。

```
systemctl reload nginx
```

完成以后，打开本地电脑上的浏览器，输入 `xiongneng.cc` 或者 `www.xiongneng.cc` 看看效果
![](https://xnstatic-1253397658.file.myqcloud.com/nginx-xnindex.png)

想再创建一个虚拟主机，可以在 conf.d 目录下面，再去创建一个 .conf 文件，
这次把这个文件命名为 xnblog.dev.conf ，你也可以在同一个配置文件里定义多个 server 配置区块去定义多个服务器，
不过把不同的服务器的配置分割成不同的文件更好一些。打开这个文件，输入下面的内容：

```nginx
server {
  listen        80;
  server_name   blog.xiongneng.cc;
  root          /usr/local/nginx/xnblog;
}
```

在`/usr/local/nginx/xnblog`下面同样创建一个`index.html`，然后写一行`this is a blog index`

测试一下 nginx 的配置 `/usr/local/nginx/sbin/nginx -t`，
再重新加载 `systemctl reload nginx`， 成功以后，打开本地电脑上的浏览器

![](https://xnstatic-1253397658.file.myqcloud.com/nginx-blog.png)

