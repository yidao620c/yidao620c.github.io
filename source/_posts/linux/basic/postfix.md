---
title: CentOS7搭建postfix邮件服务器
date: '2018-02-25 16:43:32 +0800'
toc: true
categories: [ linux ]
tags: [ linux ]
---

电子邮件系统是我们在日常工作、生活中最常用的一个网络服务。本章将首先介绍电子邮件系统的起源，
然后介绍SMTP、POP3、IMAP4等常见的电子邮件协议，
然后介绍如何在CentOS7中使用Postfix和Dovecot服务程序配置电子邮件系统服务的方法。
并结合BIND服务程序提供的DNS域名解析服务来验证客户端主机与服务器之间的邮件收发功能。
<!--more-->

## 电子邮件协议

电子邮件系统基于邮件协议来完成电子邮件的传输，常见的邮件协议有下面这些。

1. 简单邮件传输协议（Simple Mail Transfer Protocol，SMTP）：用于发送和中转发出的电子邮件，占用服务器的25/TCP端口。
2. 邮局协议版本3（Post Office Protocol 3，POP3）：用于将电子邮件存储到本地主机，占用服务器的110/TCP端口。
3. Internet消息访问协议版本4（Internet Message Access Protocol 4，IMAP4）：用于在本地主机上访问邮件，占用服务器的143/TCP端口。

在电子邮件系统中，为用户收发邮件的服务器名为邮件用户代理（Mail User Agent，MUA）。
另外，既然电子邮件系统能够让用户在离线的情况下依然可以完成数据的接收，肯定得有一个用于保存用户邮件的“信箱”服务器，
这个服务器的名字为邮件投递代理（Mail Delivery Agent，MDA），
其工作职责是把来自于邮件传输代理（Mail Transfer Agent，MTA）的邮件保存到本地的收件箱中。
其中，这个MTA的工作职责是转发处理不同电子邮件服务供应商之间的邮件，把来自于MUA的邮件转发到合适的MTA服务器。

例如，我们从新浪信箱向谷歌信箱发送一封电子邮件，这封电子邮件的传输过程如图所示。

![](https://xnstatic-1253397658.file.myqcloud.com/postfix01.png)

在生产环境中部署企业级的电子邮件系统时，有4个注意事项请留意：

1. 添加反垃圾与反病毒模块：它能够很有效地阻止垃圾邮件或病毒邮件对企业信箱的干扰。
1. 对邮件加密：可有效保护邮件内容不被黑客盗取和篡改。
1. 添加邮件监控审核模块：可有效地监控企业全体员工的邮件中是否有敏感词、是否有透露企业资料等违规行为。
1. 保障稳定性：电子邮件系统的稳定性至关重要，运维人员应做到保证电子邮件系统的稳定运行，并及时做好防范分布式拒绝服务（DDoS）攻击的准备。

一个最基础的电子邮件系统肯定要能提供发件服务和收件服务，为此需要使用基于SMTP协议的Postfix服务程序提供发件服务功能，
并使用基于POP3协议的Dovecot服务程序提供收件服务功能。这样一来，
用户就可以使用Outlook Express或Foxmail等客户端服务程序正常收发邮件了。电子邮件系统的工作流程如图所示。

![](https://xnstatic-1253397658.file.myqcloud.com/postfix02.png)

## 配置DNS服务

一般而言，我们的信箱地址类似于“root@xncoding.com”这样，也就是按照“用户名@主机地址（域名）”格式来规范的。
要想更好地检验电子邮件系统的配置效果，需要先部署bind服务程序，为电子邮件服务器和客户端提供DNS域名解析服务。

第1步：配置服务器主机名称，需要保证服务器主机名称与发信域名保持一致：

```bash
vim /etc/hostname
mail.xncoding.com
hostnamectl set-hostname mail.xncoding.com
```

第2步：清空iptables防火墙默认策略，并保存策略状态，避免因防火墙中默认存在的策略阻止了客户端DNS解析域名及收发邮件：

```bash
iptables -F
service iptables save
```

第3步：为电子邮件系统提供域名解析。

主配置文件`/etc/named.conf`：

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
        allow-transfer { key master-slave; };
...省略
}
```

区域配置文件`/etc/named.rfc1912.zones`如下：

```
zone "xncoding.com" IN {
  type master;
  file "xncoding.com.zone";
  allow-update {none;};
};
```

域名配置文件`/var/named/xncoding.com.zone`如下：

```
$TTL 1D
@       IN SOA      xncoding.com. root.xncoding.com. (
                                        0       ; serial
                                        1D      ; refresh
                                        1H      ; retry
                                        1W      ; expire
                                        3H )    ; minimum
        NS          ns.xncoding.com.
ns      IN A        192.168.1.20
        IN MX 10    mail.xncoding.com.
mail    IN A        192.168.1.20
```

然后重启DNS服务：

```bash
[root@mail ~]# systemctl restart named
[root@mail ~]# systemctl enable named
```

这样电子邮件系统所对应的服务器主机名即为`mail.xncoding.com`，而邮件域为`@xncoding.com`。
把服务器的DNS地址修改成本地IP地址。编辑文件`/etc/sysconfig/network-scripts/ifcfg-eth0`，修改如下：

```
DNS1=192.168.1.20
```

然后重启网络：

```bash
systemctl restart network
```

## 配置Postfix服务程序

Postfix是一款由IBM资助研发的免费开源电子邮件服务程序，由许多小模块组成，每个小模块都可以完成特定的功能，
因此可在生产工作环境中根据需求灵活搭配它们。

**_第1步：安装Postfix服务程序。_**

```bash
yum install -y postfix
```

**_第2步：配置Postfix服务程序。_**

配置文件`/etc/ postfix/main.cf`，大部分都是注释。几个重要的配置参数如下：

 参数              | 说明           
-----------------|--------------
 myhostname      | 邮局系统的主机名     
 mydomain        | 邮局系统的域名      
 myorigin        | 从本机发出邮件的域名名称 
 inet_interfaces | 监听的网卡接口      
 mydestination	  | 可接收邮件的主机名或域名 
 mynetworks      | 设置可转发哪些主机的邮件 
 relay_domains   | 设置可转发哪些网域的邮件 

在Postfix服务程序的主配置文件中，总计需要修改5处。首先是在第76行定义一个名为myhostname的变量，用来保存服务器的主机名称。
请大家记住这个变量的名称，下边的参数需要调用它：

```
myhostname = mail.xncoding.com
```

然后在第83行定义一个名为mydomain的变量，用来保存邮件域的名称。大家也要记住这个变量名称，下面将调用它：

```
mydomain = xncoding.com
```

在第99行调用前面的mydomain变量，用来定义发出邮件的域。调用变量的好处是避免重复写入信息，以及便于日后统一修改：

```
myorigin = $mydomain
```

第4处修改是在第116行定义网卡监听地址。可以指定要使用服务器的哪些IP地址对外提供电子邮件服务；
也可以干脆写成all，代表所有IP地址都能提供电子邮件服务：

```
inet_interfaces = all
```

最后一处修改是在第164行定义可接收邮件的主机名或域名列表。这里可以直接调用前面定义好的myhostname和mydomain变量。

```
mydestination = $myhostname, $mydomain
```

**_第3步：创建电子邮件系统的登录账户。_**

Postfix与vsftpd服务程序一样，都可以调用本地系统的账户和密码，因此在本地系统创建常规账户即可。
最后重启配置妥当的postfix服务程序，并将其添加到开机启动项中。大功告成！

```bash
[root@mail ~]# useradd xiong
[root@mail ~]# echo "xiong" | passwd --stdin xiong
更改用户 xiong 的密码 。
passwd：所有的身份验证令牌已经成功更新。
[root@mail ~]# systemctl restart postfix
[root@mail ~]# systemctl enable postfix
```

_备注_：实际生产环境邮件用户应该设置成不可登录。

## 配置Dovecot服务程序

Dovecot是一款能够为Linux系统提供IMAP和POP3电子邮件服务的开源服务程序，安全性极高，配置简单，
执行速度快，而且占用的服务器硬件资源也较少，因此是一款值得推荐的收件服务程序。

**_第1步：安装Dovecot服务程序软件包。_**

```bash
yum install -y dovecot
```

**_第2步：配置部署Dovecot服务程序。_**

修改主配置文件`/etc/dovecot/dovecot.conf`，首先是第24行，把Dovecot服务支持的电子邮件协议修改为imap、pop3和lmtp。
然后在这一行下面添加一行参数，允许用户使用明文进行密码验证。之所以这样操作，
是因为Dovecot服务程序为了保证电子邮件系统的安全而默认强制用户使用加密方式进行登录，而由于当前还没有加密系统，
因此需要添加该参数来允许用户的明文登录。

```
 24 protocols = imap pop3 lmtp
 25 disable_plaintext_auth = no
```

在主配置文件中的第48行，设置允许登录的网段地址，也就是说我们可以在这里限制只有来自于某个网段的用户才能使用电子邮件系统。
如果想允许所有人都能使用，则不用修改本参数：

```
login_trusted_networks = 192.168.1.0/24
```

**_第3步：配置邮件格式与存储路径。_**

在Dovecot服务程序单独的子配置文件中，定义一个路径，用于指定要将收到的邮件存放到服务器本地的哪个位置。
这个路径默认已经定义好了，我们只需要将该配置文件`/etc/dovecot/conf.d/10-mail.conf`中第25行前面的井号删除即可。

```
 24 #   mail_location = maildir:~/Maildir
 25 mail_location = mbox:~/mail:INBOX=/var/mail/%u
 26 #   mail_location = mbox:/var/mail/%d/%1n/%n:INDEX=/var
```

然后切换到配置Postfix服务程序时创建的账户，并在家目录中建立用于保存邮件的目录。
记得要重启Dovecot服务并将其添加到开机启动项中。至此，对Dovecot服务程序的配置部署步骤全部结束。

```bash
[root@mail ~]# su - xiong
[xiong@mail ~]$ mkdir -p mail/.imap/INBOX
[xiong@mail ~]$ exit
[root@mail ~]# systemctl restart dovecot
[root@mail ~]# systemctl enable dovecot
```

## 客户端使用电子邮件系统

如何得知电子邮件系统已经能够正常收发邮件了呢？可以使用Foxmail来进行测试。
请按照下表来设置电子邮件系统及DNS服务器和客户端主机的IP地址，以便能正常解析邮件域名。设置后的结果如图所示。

 主机            | 操作系统   | IP地址         
---------------|--------|--------------
 电子邮件系统及DNS服务器 | RHEL 7 | 192.168.1.20 
 客户端主机         | Win 10 | 192.168.1.3  

![](https://xnstatic-1253397658.file.myqcloud.com/postfix03.png)

下载安装最新版Foxmail后，打开手动设置账号信息如下：

![](https://xnstatic-1253397658.file.myqcloud.com/postfix04.png)

点击创建后验证成功，进入Foxmail主页面
![](https://xnstatic-1253397658.file.myqcloud.com/postfix05.png)

然后给`root@xncoding.com`这个账号发个邮件试试。

![](https://xnstatic-1253397658.file.myqcloud.com/postfix06.png)

成功发送邮件后，便可以在电子邮件服务器上使用mail命令查看到新邮件提醒了。如果想查看邮件的完整内容，
只需输入收件人姓名前面的编号即可。

```
[root@mail ~]# mail
Heirloom Mail version 12.5 7/5/10.  Type ? for help.
"/var/spool/mail/root": 1 message 1 new
>N  1 xiong@xncoding.com    Tue Oct  6 23:27  44/1707  "测试一封邮件"
& 1
Message  1:
From xiong@xncoding.com  Tue Oct  6 23:27:56 2020
Return-Path: <xiong@xncoding.com>
X-Original-To: root@xncoding.com
Delivered-To: root@xncoding.com
Date: Tue, 6 Oct 2020 23:27:56 +0800
From: "xiong@xncoding.com" <xiong@xncoding.com>
To: root <root@xncoding.com>
Subject: 测试一封邮件
X-Priority: 3
X-Has-Attach: no
X-Mailer: Foxmail 7.2.18.111[cn]
Content-Type: multipart/alternative;
	boundary="----=_001_NextPart063248331812_=----"
Status: R

Content-Type: text/plain;
	charset="GB2312"

我是巴拉巴拉小魔仙。
xiong@xncoding.com
& 
```

## 设置用户别名邮箱

用户别名功能是一项简单实用的邮件账户伪装技术，可以用来设置多个虚拟信箱的账户以接受发送的邮件，
从而保证自身的邮件地址不被泄露，还可以用来接收自己的多个信箱中的邮件。编辑`/etc/aliases`增加别名配置。
由于过于简单，这里不展开说明了。
