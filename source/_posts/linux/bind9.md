---
layout: post
title: CentOS7.2搭建DNS服务器
date: 2016-07-01 09:33:22 +0800
toc: true
categories: linux
tags:
  - dns
  - bind9
abbrlink: 33172
---
Bind是一款开放源码的DNS服务器软件，Bind由美国加州大学Berkeley分校开发和维护的，
全名为Berkeley Internet Name Domain它是目前世界上使用最为广泛的DNS服务器软件。

DNS服务由BIND软件提供，启动后服务名为`named`，管理工具为`rndc`，debug工具为`dig`。主要配置文件为`/etc/named.conf`。

本篇演示如何在CentOS 7上架设主域名服务器。

当局域网内主机要访问外网域名时，DNS服务器会先查本地缓存，查不到则将该解析请求转发给ISP的DNS服务器
（假设IP为`211.155.23.88`和`211.155.27.88`），而不是转发给.（root）服务器。
并且，只会对内网主机的解析请求进行转发（假设内网网段范围为`192.168.0.0/16`），而不会对外网主机的解析请求进行转发。
<!-- more -->

## 安装
这里选择将bind运行在chroot环境下面，更加安全。只需要安装`bind-chroot`即可，其他软件比如`bind`本身也会自动安装，
也即执行：
```bash
yum install -y bind-chroot
systemctl start named-chroot
systemctl enable named-chroot
systemctl status named-chroot
``

## 配置
```bash
cp -a /etc/named.conf /etc/named.conf.raw
vi /etc/named.conf
```

修改如下：
```
options {
	listen-on port 53 { any; };
	listen-on-v6 port 53 { ::1; };
	directory 	"/var/named";
	dump-file 	"/var/named/data/cache_dump.db";
	statistics-file "/var/named/data/named_stats.txt";
	memstatistics-file "/var/named/data/named_mem_stats.txt";
	allow-query     { 192.168.0.0/16; };
	recursion yes;
    allow-recursion { 192.168.0.0/16; };
    forward first;
    forwarders {
        211.155.23.88;
        211.155.27.88;
    };

	dnssec-enable no;
	dnssec-validation no;
    dnssec-lookaside no;

	/* Path to ISC DLV key */
	bindkeys-file "/etc/named.iscdlv.key";

	managed-keys-directory "/var/named/dynamic";

	pid-file "/run/named/named.pid";
	session-keyfile "/run/named/session.key";
};

logging {
        channel default_debug {
                file "data/named.run";
                severity dynamic;
        };
};

zone "." IN {
	type hint;
	file "named.ca";
};

/*新增xn.local的域名解析*/
zone "xn.local" IN {
    type master;
    file "fwd.xn.local.db";
    allow-update { none; };
    allow-query { any; };
};

include "/etc/named.rfc1912.zones";
include "/etc/named.root.key";
```

新增一个文件 `vi /var/named/fwd.xn.local.db`，内容如下：
```
$TTL 86400
@ IN SOA primary.xn.local. root.xn.local. (
2016042112 ;Serial
3600 ;Refresh
1800 ;Retry
604800 ;Expire
43200 ;Minimum TTL
)
;Name Server Information
@ IN NS primary.xn.local.

;IP address of Name Server
primary IN A 192.168.217.161

;Mail exchanger
xn.local. IN MX 10 mail.xn.local.

;A - Record HostName To Ip Address
www IN A 192.168.217.161
mail IN A 192.168.217.161

;CNAME record
ftp IN CNAME www.xn.local.
```


最后启动`named`服务：
```bash
systemctl start named
systemctl enable named
systemctl status named
```

## 测试与验证
默认情况下，DNS服务的日志信息会放置到`/var/log/messages`文档中。如果有修改配置文件，
并启动或重启named服务的话，建议第一时间先查看这个日志文档，看有没有报错：

![](https://xnstatic-1253397658.file.myqcloud.com/dns01.png)

检查DNS服务的端口（端口53）是否有开启：

![](https://xnstatic-1253397658.file.myqcloud.com/dns02.png)

## 指定DNS服务器
CentOS7指定DNS服务器方法：
```bash
systemctl restart NetworkManager.service
nmcli connection show
nmcli con mod eth0 ipv4.dns "192.168.217.161"
# nmcli con mod eth0 ipv4.dns "192.168.217.161 8.8.8.8"
nmcli con up eth0
```

利用`dig`工具测试主DNS能否正常解析外网网址：
```
# 如果没安装dig命令，可以使用下面命令安装
yum install bind-utils
dig @192.168.217.161 www.baidu.com
```

![](https://xnstatic-1253397658.file.myqcloud.com/dns02.png)

搞定！

