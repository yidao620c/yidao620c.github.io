---
title: 5分钟搭建一个IDEA服务
date: 2016-10-16 20:22:22 +0800
toc: true
categories: [ 开发工具 ]
tags: [ 开发工具 ]
---

作为一个码农对于那些优秀的开发工具爱不释手，它们对框架的支持、界面、插件都是那么的优秀，
大大加快了开发的速度以及开发的乐趣，酷炫的界面也能大大的装一个逼。写到这里大家应该能猜到我说的是啥了，
没错，就是JetBrains出品的全系列IDE开发工具，比如IntelliJ IDEA、PyCharm等等。

对于暂时经济不宽裕的同学，比较明智的选择是Google一台active服务器即可，
有能力的同学不妨尝试自行架设，这也就是本文的目的啦。

喝水不忘挖井人，在此向服务器软件的作者Lanyu表示衷心的感谢。
<!-- more -->

点击→ [Lanyu的博客](http://blog.lanyus.com/)

## 服务器

在上面的博客页面有下载地址，支持很多的操作系统平台，这里我选的是linux64位的。
更新这篇文章的时候最新版是v1.6版。

接下来，介绍如何部署到Linux服务器上，首先将xxxServer_linux_amd64上传到任意目录，
我这里是 `/root/work/` 目录，先将名字改了，太长了

```bash
mv xxxServer_linux_amd64 xnServer
```

接下来 需要把它运行起来，先加一个可执行权限

```bash
chmod +x xnServer
```

运行

```bash
/root/work/xnServer -p 1027 -u yidao -prolongationPeriod 999999999 -l 127.0.0.1
```

默认运行会出现以下信息，则为成功(省略)。如果要后台运行，请使用nohup命令。

启动参数说明：

1. -l 指定绑定监听到哪个IP
2. -u 用户名参数，当未设置-u参数，且计算机用户名为^[a-zA-Z0-9]+$时，使用计算机用户名作为idea用户名
3. -p 参数，用于指定监听的端口
4. -prolongationPeriod 指定过期时间参数

## 守护进程

如果你使用的是CentOS7，直接用systemd方式。

新建一个服务配置`/etc/systemd/system/xnserver.service`，内容如下：

```
[Unit]
Description=XnServer
After=syslog.target network.target

[Service]
Type=simple

ExecStart=/root/work/xnServer -u yidao -prolongationPeriod 999999999 -l 127.0.0.1 &
ExecStop=/bin/kill -HUP $MAINPID

User=root
Group=root

[Install]
WantedBy=multi-user.target
```

保存之后，执行`systemctl enable xnserver.service`，开机即可启动。

然后也可以手动启动：`systemctl start xnserver`

## nginx配置

接下来，将自己的域名采用nginx反向代理过来。

首先去域名解析那儿添加一个子域名`active.xncoding.com`，然后修改`/etc/nginx/nginx.conf`配置：

```
server {
    listen 80;
    server_name active.xncoding.com;
    root /var/www/html/;

    location / {
        proxy_pass http://127.0.0.1:1027;
        proxy_redirect off;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
    access_log off; #access_log end
    error_log /dev/null; #error_log end
}
```

重启nginx `service nginx restart`

然后激活服务器地址就填`http://active.xncoding.com`即可。

如果嫌nginx麻烦，也不需要配置，直接用ip地址加端口号也是一样：`http://xx.xx.xx.xx:1027/`

## IDE设置

点击 `Help` -> `Register..` -> 选中`Licence Server`，然后填入上面的激活地址即可。

