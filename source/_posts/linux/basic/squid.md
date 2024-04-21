---
title: CentOS7.2搭建代理服务器
date: 2016-07-07 21:15:16 +0800
toc: true
categories: [ linux ]
tags: [ linux ]
---

有时候内网很多机器都不能上外网，只能开放几个特定ip访问外网，
那么可以在这个可以上外网的机器上面搭建代理服务器，其他机器配置好代理就能上网了。

不管是测试用途还是自己使用，squid都是一个很不错的代理工具。支持正向代理、反向代理、还有透明代理。
本篇演示搭建了一个简单的squid的正向代理，同时支持认证，随便记记笔记。
<!-- more -->

## 安装

```bash
yum install squid -y
yum install httpd-tools -y
```

## 生成密码文件

```
mkdir /etc/squid3/
# xiongneng 是用户名
htpasswd -cd /etc/squid3/passwords xiongneng
# 提示输入密码，在这里我设的密码为 123456
# 注意密码不要超过8位
```

## 测试密码文件

```
/usr/lib64/squid/basic_ncsa_auth /etc/squid3/passwords
# 输入 用户名 密码
xiongneng 123456
# 提示OK说明成功，ERR是有问题，请检查一下之前步骤
OK

# 测试完成，crtl + c 打断
```

## 配置

```
vim /etc/squid/squid.conf

# 在最后添加

auth_param basic program /usr/lib64/squid/basic_ncsa_auth /etc/squid3/passwords
auth_param basic realm proxy
acl authenticated proxy_auth REQUIRED
http_access allow authenticated

# 这里是端口号，可以按需修改
# http_port 3128 这样写会同时监听ipv6和ipv4的端口，推荐适应下面的配置方法。
http_port 0.0.0.0:3128
```

## 权限控制

squid的权限控制很灵活，具体配置方法可以参考 [官方文档](http://www.squid-cache.org/Doc/config/acl/)，
或者 [Squid中文权威指南](http://home.arcor.de/pangj/squid/chap06.html)，
具体工作原理有点像iptables，用规则去卡控流量。默认的配置只能允许内网用户访问，如果有更多需求，你还可以指定很多规则！

下面是我的配置实例：

```
# Example rule allowing access from your local networks.
# Adapt to list your (internal) IP networks from where browsing
# should be allowed
acl localnet src 10.0.0.0/8	# RFC1918 possible internal network
acl localnet src 172.16.0.0/12	# RFC1918 possible internal network
acl localnet src 192.168.0.0/16	# RFC1918 possible internal network
acl localnet src fc00::/7       # RFC 4193 local private network range
acl localnet src fe80::/10      # RFC 4291 link-local (directly plugged) machines

acl SSL_ports port 443
acl Safe_ports port 80		# http
acl Safe_ports port 21		# ftp
acl Safe_ports port 443		# https
acl Safe_ports port 70		# gopher
acl Safe_ports port 210		# wais
acl Safe_ports port 1025-65535	# unregistered ports
acl Safe_ports port 280		# http-mgmt
acl Safe_ports port 488		# gss-http
acl Safe_ports port 591		# filemaker
acl Safe_ports port 777		# multiling http
acl CONNECT method CONNECT

#
# Recommended minimum Access Permission configuration:
#
# Deny requests to certain unsafe ports
http_access deny !Safe_ports

# Deny CONNECT to other than secure SSL ports
http_access deny CONNECT !SSL_ports

# Only allow cachemgr access from localhost
http_access allow localhost manager
http_access deny manager

# We strongly recommend the following be uncommented to protect innocent
# web applications running on the proxy server who think the only
# one who can access services on "localhost" is a local user
#http_access deny to_localhost

#
# INSERT YOUR OWN RULE(S) HERE TO ALLOW ACCESS FROM YOUR CLIENTS
#

# Example rule allowing access from your local networks.
# Adapt localnet in the ACL section to list your (internal) IP networks
# from where browsing should be allowed
http_access allow localnet
http_access allow localhost

# And finally deny all other access to this proxy
http_access deny all
```

## 日志

squid的日志默认是打开的，位于目录`/var/log/squid/`，当然这个地址还有日志的格式都是可以完全自定义的

```
[root@controller161 ~]# ll /var/log/squid/
total 496
-rw-r----- 1 squid squid 355208 May  6 12:17 access.log
-rw-r----- 1 squid squid   1846 Jul 10  2016 access.log-20160710.gz
-rw-r----- 1 squid squid   3710 Jul 15  2016 access.log-20160718.gz
-rw-r----- 1 squid squid 125341 May  4 15:19 cache.log
-rw-r----- 1 squid squid   1325 Jul  9  2016 cache.log-20160710.gz
-rw-r----- 1 squid squid   1110 Jul 14  2016 cache.log-20160718.gz
```

## 启动服务

```bash
# 开启启动
systemctl enable squid.service
# 启动
systemctl start squid.service
# 停止
systemctl stop squid.service
# 重启
systemctl restart squid.service
```

## 代理服务器设置

在其他CentOS机器上面配置各种代理方法

### 全局代理

`vim /etc/profile`，在最后加入

```bash
export http_proxy="http://username:password@proxy_ip:port"
export https_proxy="http://username:password@proxy_ip:port"
```

### yum代理设置

编辑`/etc/yum.conf`，在最后加入

```
# Proxy
proxy=http://username:password@proxy_ip:port/
```

### wget的代理设置

编辑`/etc/wgetrc`，在最后加入

```
# Proxy
http_proxy=http://username:password@proxy_ip:port/
https_proxy=http://username:password@proxy_ip:port/
ftp_proxy=http://username:password@proxy_ip:port/
```

### curl的代理设置

在`~/.bashrc`里面增加一个别名:

```bash
alias curl="curl -x http://username:password@proxy_ip:port"
```

另外一种方法是编辑`~/.curlrc`文件 (没有就创建一个):

```
proxy = http://username:password@proxy_ip:port
```

