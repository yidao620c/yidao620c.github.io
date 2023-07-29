---
title: mqtt消息中间件mosquitto的安装和配置
date: 2015-05-17 16:33:07 +0800
toc: true
categories: [ 中间件 ]
tags: [ mqtt ]
---

Mosquitto是一个开源(BSD许可证)的消息代理，实现MQTT(消息队列遥测传输)协议版本3.1.1。

MQTT（MQ Telemetry Transport），消息队列遥测传输协议，轻量级的发布/订阅协议，
适用于一些条件比较苛刻的环境，进行低带宽、不可靠或间歇性的通信。目前已经是物联网消息通信事实上的标准协议了。

值得一提的是mqtt提供三种不同质量的消息服务：

* "至多一次"：消息发布完全依赖底层 TCP/IP 网络。会发生消息丢失或重复。这一级别可用于如下情况，环境传感器数据，丢失一次读记录无所谓，因为不久后还会有第二次发送。
* "至少一次"：确保消息到达，但消息重复可能会发生。
* "只有一次"：确保消息到达一次。这一级别可用于如下情况，在计费系统中，消息重复或丢失会导致不正确的结果。

<!-- more -->

### 安装

参考官网 <http://mosquitto.org/download/>

服务器操作系统为CentOS6.4，使用最简单的yum安装

#### 1，先加入yum源：

在`/etc/yum.repos.d/`目录中新建一个`mosquitto.repo`文件，里面写入：

```
[home_oojah_mqtt]
name=mqtt (CentOS_CentOS-6)
type=rpm-md
baseurl=http://download.opensuse.org/repositories/home:/oojah:/mqtt/CentOS_CentOS-6/
gpgcheck=1
gpgkey=http://download.opensuse.org/repositories/home:/oojah:/mqtt/CentOS_CentOS-6/repodata/repomd.xml.key
enabled=1
```

#### 2，开始安装

```bash
sudo yum search all mosquitto
sudo yum install mosquitto mosquitto-clients libmosquitto-devel libmosquittopp-devel python-mosquitto
```

#### 3，配置

安装完成之后，所有配置文件会被放置于/etc/mosquitto/目录下，
其中最重要的就是Mosquitto的配置文件，即mosquitto.conf，以下是详细的配置参数说明。

```
# =================================================================
# General configuration
# =================================================================

# 客户端心跳的间隔时间
#retry_interval 20

# 系统状态的刷新时间
#sys_interval 10

# 系统资源的回收时间，0表示尽快处理
#store_clean_interval 10

# 服务进程的PID
#pid_file /var/run/mosquitto.pid

# 服务进程的系统用户
#user mosquitto

# 客户端心跳消息的最大并发数
#max_inflight_messages 10

# 客户端心跳消息缓存队列
#max_queued_messages 100

# 用于设置客户端长连接的过期时间，默认永不过期
#persistent_client_expiration

# =================================================================
# Default listener
# =================================================================

# 服务绑定的IP地址
#bind_address

# 服务绑定的端口号
#port 1883

# 允许的最大连接数，-1表示没有限制
#max_connections -1

# cafile：CA证书文件
# capath：CA证书目录
# certfile：PEM证书文件
# keyfile：PEM密钥文件
#cafile
#capath
#certfile
#keyfile

# 必须提供证书以保证数据安全性
#require_certificate false

# 若require_certificate值为true，use_identity_as_username也必须为true
#use_identity_as_username false

# 启用PSK（Pre-shared-key）支持
#psk_hint

# SSL/TSL加密算法，可以使用"openssl ciphers"命令获取
# as the output of that command.
#ciphers

# =================================================================
# Persistence
# =================================================================

# 消息自动保存的间隔时间
#autosave_interval 1800

# 消息自动保存功能的开关
#autosave_on_changes false

# 持久化功能的开关
persistence true

# 持久化DB文件
#persistence_file mosquitto.db

# 持久化DB文件目录
#persistence_location /var/lib/mosquitto/

# =================================================================
# Logging
# =================================================================

# 4种日志模式：stdout、stderr、syslog、topic
# none 则表示不记日志，此配置可以提升些许性能
log_dest none

# 选择日志的级别（可设置多项）
#log_type error
#log_type warning
#log_type notice
#log_type information

# 是否记录客户端连接信息
#connection_messages true

# 是否记录日志时间
#log_timestamp true

# =================================================================
# Security
# =================================================================

# 客户端ID的前缀限制，可用于保证安全性
#clientid_prefixes

# 允许匿名用户
#allow_anonymous true

# 用户/密码文件，默认格式：username:password
#password_file

# PSK格式密码文件，默认格式：identity:key
#psk_file

# pattern write sensor/%u/data
# ACL权限配置，常用语法如下：
# 用户限制：user <username>
# 话题限制：topic [read|write] <topic>
# 正则限制：pattern write sensor/%u/data
#acl_file

# =================================================================
# Bridges
# =================================================================

# 允许服务之间使用"桥接"模式（可用于分布式部署）
#connection <name>
#address <host>[:<port>]
#topic <topic> [[[out | in | both] qos-level] local-prefix remote-prefix]

# 设置桥接的客户端ID
#clientid

# 桥接断开时，是否清除远程服务器中的消息
#cleansession false

# 是否发布桥接的状态信息
#notifications true

# 设置桥接模式下，消息将会发布到的话题地址
# $SYS/broker/connection/<clientid>/state
#notification_topic

# 设置桥接的keepalive数值
#keepalive_interval 60

# 桥接模式，目前有三种：automatic、lazy、once
#start_type automatic

# 桥接模式automatic的超时时间
#restart_timeout 30

# 桥接模式lazy的超时时间
#idle_timeout 60

# 桥接客户端的用户名
#username

# 桥接客户端的密码
#password

# bridge_cafile：桥接客户端的CA证书文件
# bridge_capath：桥接客户端的CA证书目录
# bridge_certfile：桥接客户端的PEM证书文件
# bridge_keyfile：桥接客户端的PEM密钥文件
#bridge_cafile
#bridge_capath
#bridge_certfile
#bridge_keyfile

# 自己的配置可以放到以下目录中
include_dir /etc/mosquitto/conf.d
```

#### 4，启动服务，两种方式

```bash
mosquitto -c /etc/mosquitto/mosquitto.conf -d
sudo /etc/init.d/mosquitto start
```

------------

### 演示部分

前面已经开启了服务，如果没有请参考前面步骤。

一、开启另一个终端窗口，运行订阅程序mosquitto_sub:

*注意*：

消息推送的发布和订阅要有主题，选项[-t] 主题，即：mosquitto -t 主题

如需指定用户名称则加选项[-i] 用户名，即：mosquitto_sub -t 主题 -i 订阅端

```bash
mosquitto_sub -t mqtt
```

二、开启另一个终端窗口，运行发布程序mosquitto_pub:

指定消息推送的主题，发布端用户名和消息：

```bash
mosquitto_pub -t 主题 -i 发布端 -h 主机 -m 你好
```

*注意：如果消息中间有空格则消息要用引号括起来。

```bash
mosquitto_pub -t 主题 -i 发布端 -h host -m '我是发布端，你好。'
mosquitto_pub -h localhost -t mqtt -m "hello world."
```

这时候前面那个订阅窗口就可以收到"hello world"的消息了。

演示成功！！！

