---
title: 使用supervisor管理进程
date: 2016-10-12 08:22:22 +0800
toc: true
categories: [ linux ]
tags: [ linux ]
---

Supervisor (http://supervisord.org) 是一个用 Python 写的进程管理工具，
可以很方便的用来启动、重启、关闭进程（不仅仅是Python进程）。除了对单个进程的控制，还可以同时启动、关闭多个进程，
有时候服务器出问题导致所有应用程序都被杀死，此时可以用supervisor同时启动所有应用程序。
<!-- more -->

## 组成部分

supervisor 主要由两部分组成：

1. supervisord(server 部分)：主要负责管理子进程，响应客户端命令以及日志的输出等；
2. supervisorctl(client 部分)：命令行客户端，用户可以通过它与不同的 supervisord 进程联系，获取子进程的状态等。

## 安装

可以直接使用 pip 安装：

```bash
sudo pip install supervisor
```

如果是 Ubuntu 系统，也可以使用 apt-get 来安装

安装完成之后，可以运行 echo_supervisord_conf 生成默认的配置文件：

```bash
echo_supervisord_conf > /etc/supervisord.conf
```

## supervisor自启动服务

这里只介绍在CentOS7上面添加supervisord服务为自启动，其他平台请自行google

修改 ``/lib/systemd/system/supervisord.service`，添加如下内容：

```
# supervisord service for systemd (CentOS 7.0+)
[Unit]
Description=Supervisor daemon

[Service]
Type=forking
ExecStart=/usr/bin/supervisord -c /etc/supervisord.conf
ExecStop=/usr/bin/supervisorctl $OPTIONS shutdown
ExecReload=/usr/bin/supervisorctl $OPTIONS reload
KillMode=process
Restart=on-failure
RestartSec=60s

[Install]
WantedBy=multi-user.target
```

无需修改/etc/supervisord.conf配置文件，该启动脚本都能够添加到systemctl自启动服务:

```bash
systemctl enable supervisord.service
systemctl start/restart/stop supervisord.service
```

## 配置详解

Supervisor 相当强大，提供了很丰富的功能，不过我们可能只需要用到其中一小部分。安装完成之后，可以编写配置文件，来满足自己的需求。
为了方便，我们把配置分成两部分：

1. supervisord（这是 server 端，对应的有 client 端：supervisorctl）
2. 应用程序（即我们要管理的程序）。

首先来看 supervisord 的配置文件。去除里面大部分注释和"不相关"的部分，我们可以先看这些配置：

```
[unix_http_server]
file=/tmp/supervisor.sock   ; UNIX socket 文件，supervisorctl 会使用
;chmod=0700                 ; socket 文件的 mode，默认是 0700
;chown=nobody:nogroup       ; socket 文件的 owner，格式： uid:gid

;[inet_http_server]         ; HTTP 服务器，提供 web 管理界面
;port=127.0.0.1:9001        ; Web 管理后台运行的 IP 和端口，如果开放到公网，需要注意安全性
;username=user              ; 登录管理后台的用户名
;password=123               ; 登录管理后台的密码

[supervisord]
logfile=/tmp/supervisord.log ; 日志文件，默认是 $CWD/supervisord.log
logfile_maxbytes=50MB        ; 日志文件大小，超出会 rotate，默认 50MB
logfile_backups=10           ; 日志文件保留备份数量默认 10
loglevel=info                ; 日志级别，默认 info，其它: debug,warn,trace
pidfile=/tmp/supervisord.pid ; pid 文件
nodaemon=false               ; 是否在前台启动，默认是 false，即以 daemon 的方式启动
minfds=1024                  ; 可以打开的文件描述符的最小值，默认 1024
minprocs=200                 ; 可以打开的进程数的最小值，默认 200

; the below section must remain in the config file for RPC
; (supervisorctl/web interface) to work, additional interfaces may be
; added by defining them in separate rpcinterface: sections
[rpcinterface:supervisor]
supervisor.rpcinterface_factory = supervisor.rpcinterface:make_main_rpcinterface

[supervisorctl]
serverurl=unix:///tmp/supervisor.sock ; 通过 UNIX socket 连接 supervisord，路径与 unix_http_server 部分的 file 一致
;serverurl=http://127.0.0.1:9001 ; 通过 HTTP 的方式连接 supervisord

; 包含其他的配置文件
[include]
files = relative/directory/*.ini    ; 可以是 *.conf 或 *.ini
```

然后可以通过 supervisord 命令启动 supervisord:

```bash
supervisord -c /etc/supervisord.conf
```

查看 supervisord 是否在运行：

```bash
ps aux | grep supervisord
```

我们可以看到 supervisord 已经被启动了， 然后进入 supervisorctl 的 shell 界面:

```bash
$ supervisorctl
supervisor> status
supervisor>
```

由于目前没有添加任何需要管理的进程，所以 status 没有输出人和结果，
接下来我们添加一个需要管理的进程 (以启动一个 celery 的 worker 为例)：

```
[program:celeryd]
user=tomcat                                          ; 用哪个用户启动
command=celery worker --app=task -l info             ; 启动命令
stdout_logfile=/var/log/supervisor/celeryd_out.log   ; stdout 日志输出位置
stderr_logfile=/var/log/supervisor/celeryd_err.log   ; stderr 日志输出位置
stdout_logfile_maxbytes=20MB                         ; stdout 日志文件大小，默认 50MB
stdout_logfile_backups=20                            ; stdout 日志文件备份数
autostart=true                                       ; 在 supervisord 启动的时候自动启动
autorestart=true                                     ; 程序异常退出后自动重启
startsecs=10                                         ; 启动 10 秒后没有异常退出，就当作已经正常启动
```

一份配置文件至少需要一个 [program:x] 部分的配置，来告诉 supervisord 需要管理那个进程。
[program:x] 语法中的 x 表示 program name，会在客户端（supervisorctl 或 web 界面）显示，
在 supervisorctl 中通过这个值来对程序进行 start、restart、stop 等操作。

## 使用 supervisorctl

然后运行以下命令更新配置并启动进程：

```bash
$ supervisorctl reread (只更新配置文件)
celeryd: available

$ supervisorctl update (只启动有改动的进程)
celeryd: added process group

$ supervisorctl status
celeryd                          RUNNING    pid 1919, uptime 0:00:18
```

我们看到 celery worker 已经被成功启动了。你可以使用不同的命令来控制进程的启动和关闭:

```bash
$ supervisorctl stop celeryd
celeryd: stopped
$ supervisorctl start celeryd
celeryd: started
$ supervisorctl restart celeryd
celeryd: stopped
celeryd: started
```

把所有的配置文件都放在 supervisord.conf 并不是个好主意，一旦管理的进程过多，就很麻烦。
所以一般都会 新建一个目录来专门放置进程的配置文件，然后通过 include 的方式来获取这些配置信息:

```bash
[include]
files = /etc/supervisor/conf.d/*.conf
```

然后在目录 /etc/supervisor/conf.d 下新建一个配置文件 celery.conf, 配置信息与上面的一致，效果是一样的。

## 命令详解

1. supervisord: 初始启动Supervisord，启动、管理配置中设置的进程;
2. supervisorctl stop(start, restart) xxx/all，停止（启动，重启）某一个进程(xxx)/全部;
3. supervisorctl reread: 只载入最新的配置文件, 并不重启任何进程;
4. supervisorctl reload: 载入最新的配置文件，停止原来的所有进程并按新的配置启动管理所有进程;
5. supervisorctl update: 根据最新的配置文件，启动新配置或有改动的进程，配置没有改动的进程不会受影响而重启;

## 其他

除了 supervisorctl 之外，还可以配置 supervisrod 启动 web 管理界面，这个 web 后台使用 Basic Auth 的方式进行身份认证。

除了单个进程的控制，还可以配置 group，进行分组管理。

经常查看日志文件，包括 supervisord 的日志和各个 pragram 的日志文件，
程序 crash 或抛出异常的信息一半会输出到 stderr，可以查看相应的日志文件来查找问题。

Supervisor有很丰富的功能，还有其他很多配置，可以在官方文档获取更多信息：http://supervisord.org/index.html

## FAQ

最后记录一下问题和解决办法:

reload的时候报错

```
No such file or directory: file: /usr/lib64/python2.7/socket.py
```

原因是没有指定program的执行用户，指定即可`user=root`

status命令的时候报错

```
unix:///tmp/supervisor.sock no such file
```

原因是supervisord进程根本就没启动，请重新执行启动命令。

```
supervisord -c /etc/supervisord.conf
```

这时候启动又报错

```
/var/log/supervisor/xxx_out.log does not exist
```

原因是`/var/log/supervisor`目录不存在，去mkdir一个就可以了。

```
mkdir /var/log/supervisor
```

