---
layout: post
title: CentOS7.2搭建DNS服务器
date: 2016-07-01 09:33:22 +0800
toc: true
categories: linux
tags:
  - dns
  - bind9
  - linux
abbrlink: 33172
---
作为互联网基础设施中重要一环的DNS域名解析服务，在互联网中所承担的重要角色和发挥的重要作用。
Bind是一款开放源码的DNS服务器软件，Bind由美国加州大学Berkeley分校开发和维护的，
全名为`Berkeley Internet Name Domain`，它是目前世界上使用最为广泛的DNS服务器软件。

本篇演示如何在CentOS 7上架设域名服务器，部署DNS从服务器以及DNS缓存服务器来提升用户的域名查询体验。
以及如何使用chroot牢笼机制插件来保障bind服务程序的可靠性，
并向大家演示如何在主服务器与从服务器之间部署TSIG密钥加密功能，来进一步保障迭代查询中数据的安全性。
<!-- more -->

## DNS域名解析服务简介
相较于由数字构成的IP地址，域名更容易被理解和记忆，所以我们通常更习惯通过域名的方式来访问网络中的资源。
DNS用于管理和解析域名与IP地址对应关系的技术，简单来说，就是能够接受用户输入的域名或IP地址，
然后自动查找与之匹配（或者说具有映射关系）的IP地址或域名，即将域名解析为IP地址（正向解析），
或将IP地址解析为域名（反向解析）。

DNS域名解析服务采用了类似目录树的层次结构来记录域名与IP地址之间的对应关系，从而形成了一个分布式的数据库系统。
提供了下面三种类型的服务器。

1. 主服务器：在特定区域内具有唯一性，负责维护该区域内的域名与IP地址之间的对应关系。
1. 从服务器：从主服务器中获得域名与IP地址的对应关系并进行维护，以防主服务器宕机等情况。
1. 缓存服务器：通过向其他域名解析服务器查询获得域名与IP地址的对应关系，并将经常查询的域名信息保存到服务器本地，
以此来提高重复查询时的效率。

简单来说，主服务器是用于管理域名和IP地址对应关系的真正服务器，从服务器帮助主服务器“打下手”，分散部署在各个国家、省市或地区，
以便让用户就近查询域名，从而减轻主服务器的负载压力。缓存服务器不太常用，一般部署在企业内网的网关位置，用于加速用户的域名查询请求。

## 安装
这里选择将bind运行在chroot环境下面，更加安全。只需要安装`bind-chroot`即可，其他软件比如`bind`本身也会自动安装。
```bash
yum install -y bind-chroot
```

## 配置
bind服务程序的配置并不简单，因为要想为用户提供健全的DNS查询服务，要在本地保存相关的域名数据库，
而如果把所有域名和IP地址的对应关系都写入到某个配置文件中，估计要有上千万条的参数，这样既不利于程序的执行效率，
也不方便日后的修改和维护。因此在bind服务程序中有下面这三个比较关键的文件。

    主配置文件（/etc/named.conf）：这些参数用来定义bind服务程序的运行。
    区域配置文件（/etc/named.rfc1912.zones）：用来保存域名和IP地址对应关系的所在位置。当需要查看或修改时，可根据这个位置找到相关文件。
    数据配置文件目录（/var/named）：该目录用来保存域名和IP地址真实对应关系的数据配置文件。

在Linux系统中，bind服务程序的名称为named。首先需要在/etc目录中找到该服务程序的主配置文件，修改`listen-on`和`allow-query`的值为`any`，
分别表示服务器上的所有IP地址均可提供DNS域名解析服务，以及允许所有人对本服务器发送DNS查询请求。这两个地方一定要修改准确。
```
options {
        listen-on port 53 { any; };
        listen-on-v6 port 53 { ::1; };
        directory       "/var/named";
        dump-file       "/var/named/data/cache_dump.db";
        statistics-file "/var/named/data/named_stats.txt";
        memstatistics-file "/var/named/data/named_mem_stats.txt";
        recursing-file  "/var/named/data/named.recursing";
        secroots-file   "/var/named/data/named.secroots";
        allow-query     { any; };
}
```

bind服务程序的区域配置文件（/etc/named.rfc1912.zones）用来保存域名和IP地址对应关系的所在位置。在这个文件中，
定义了域名与IP地址解析规则保存的文件位置以及服务类型等内容，而没有包含具体的域名、IP地址对应关系等信息。
服务类型有三种，分别为hint（根区域）、master（主区域）、slave（辅助区域），其中常用的master和slave指的就是主服务器和从服务器。
将域名解析为IP地址的正向解析参数和将IP地址解析为域名的反向解析参数分别如下所示。

_正向解析配置例子：_
```
zone "xncoding.com" IN {
  type master;  #服务类型
  file "xncoding.com.zone";  #域名与IP解析规则保存的文件位置
  allow-update {none;};  #运行哪些客户端动态更新解析信息
};
```

_反向解析配置例子：_
```
zone "1.168.192.in-addr.arpa" IN {  #表示192.168.1.0/24网段的反向解析区域
  type master; #服务类型
  file "192.168.1.arpa";
};
```

在修改配置文件过程中，可执行`named-checkconf`命令和`named-checkzone`命令，分别检查主配置文件与数据配置文件中语法或参数的错误。

### 正向解析配置

第1步：编辑区域配置文件`/etc/named.rfc1912.zones`。该文件中默认已经有了一些无关紧要的解析参数，旨在让用户有一个参考。
我们可以将下面的参数添加到区域配置文件的最下面，
当然，也可以将该文件中的原有信息全部清空，而只保留自己的域名解析信息：
```
zone "xncoding.com" IN {
  type master;
  file "xncoding.com.zone";
  allow-update {none;};
};
```

第2步：编辑数据配置文件。我们可以从`/var/named`目录中复制一份正向解析的模板文件`named.localhost`，
然后把域名和IP地址的对应数据填写数据配置文件中并保存。在复制时记得加上-a参数，这可以保留原始文件的所有者、所属组、权限属性等信息，
以便让bind服务程序顺利读取文件内容：
```bash
[root@master ~]# cd /var/named/
[root@master named]# ls -al named.localhost
-rw-r-----. 1 root named 152 6月  21 2007 named.localhost
[root@master named]# cp -a named.localhost xncoding.com.zone
```
然后编辑文件`xncoding.com.zone`如下所示：
```
$TTL 1D #生存周期为1天
@       IN SOA      xncoding.com. root.xncoding.com. (
        #授权信息开始 #DNS区域的地址  #域名管理员的邮箱(不要用@符号)
                                        0       ; serial   #更新序列号
                                        1D      ; refresh  #更新时间
                                        1H      ; retry    #重试延时
                                        1W      ; expire   #失效时间
                                        3H )    ; minimum  #无效解析记录的缓存时间
        NS          ns.xncoding.com.    #域名服务器记录
ns      IN A        192.168.1.20        #地址记录(ns.xncoding.com.)
        IN MX 10    mail.xncoding.com.  #邮箱交换记录
mail    IN A        192.168.1.20        #地址记录(mail.xncoding.com.)
www     IN A        192.168.1.20        #地址记录(www.xncoding.com.)
bbs     IN A        192.168.1.21        #地址记录(bbs.xncoding.com.)
```

然后重启named服务：
```bash
systemctl restart named
```

第3步：检验解析结果。为了检验解析结果，一定要先把Linux系统网卡中的DNS地址参数修改成本机IP地址，
这样就可以使用由本机提供的DNS查询服务了。nslookup命令用于检测能否从DNS服务器中查询到域名与IP地址的解析记录，
进而更准确地检验DNS服务器是否已经能够为用户提供服务。

编辑`vim /etc/sysconfig/network-scripts/ifcfg`，将DNS1的值改成本机IP。然后重启网络：
```bash
systemctl restart network
```

使用nslookup命令来测试DNS解析：
```bash
yum install -y bind-utils
[root@master named]# nslookup
> www.xncoding.com
Server:		192.168.1.20
Address:	192.168.1.20#53

Name:	www.xncoding.com
Address: 192.168.1.20
> bbs.xncoding.com
Server:		192.168.1.20
Address:	192.168.1.20#53

Name:	bbs.xncoding.com
Address: 192.168.1.21
```

###  反向解析配置
在DNS域名解析服务中，反向解析的作用是将用户提交的IP地址解析为对应的域名信息。

第1步：编辑区域配置文件`/etc/named.rfc1912.zones`。在编辑该文件时，除了不要写错格式之外，还需要记住此处定义的数据配置文件名称，
因为一会儿还需要在`/var/named`目录中建立与其对应的同名文件。反向解析是把IP地址解析成域名格式，
因此在定义zone（区域）时应该要把IP地址反写，比如原来是`192.168.1.0`，反写后应该就是`1.168.192`，而且只需写出IP地址的网络位即可。
把下列参数添加至正向解析参数的后面。
```
zone "1.168.192.in-addr.arpa" IN {
  type master;
  file "192.168.1.arpa";
};
```

第2步：编辑数据配置文件。首先从`/var/named`目录中复制一份反向解析的模板文件`named.loopback`，然后把下面的参数填写到文件中。
其中，IP地址仅需要写主机位。
```
$TTL 1D
@       IN SOA  xncoding.com.   root.xncoding.com. (
                                        0       ; serial
                                        1D      ; refresh
                                        1H      ; retry
                                        1W      ; expire
                                        3H )    ; minimum
        NS      ns.xncoding.com.
ns      A       192.168.1.20
20      PTR     ns.xncoding.com.   #PTR为指针记录，仅用于反向解析中
20      PTR     www.xncoding.com.
20      PTR     mail.xncoding.com.
21      PTR     bbs.xncoding.com.
```

重启named服务后验证：
```bash
[root@master named]# systemctl restart named
[root@master named]# nslookup 
> 192.168.1.20
20.1.168.192.in-addr.arpa	name = mail.xncoding.com.
20.1.168.192.in-addr.arpa	name = ns.xncoding.com.
20.1.168.192.in-addr.arpa	name = www.xncoding.com.
> 192.168.1.21
21.1.168.192.in-addr.arpa	name = bbs.xncoding.com.
> 
```

## 部署从服务器



