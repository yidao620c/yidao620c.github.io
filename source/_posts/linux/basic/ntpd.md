---
title: CentOS7搭建NTP服务器
date: 2018-02-16 12:55:13 +0800
toc: true
categories: [ linux ]
tags: [ linux ]
---

NTP是网络时间协议(Network Time Protocol)，它是用来同步网络中各个计算机的时间的协议。把计算机的时钟同步到世界协调时UTC，
其精度在局域网内可达0.1ms，在互联网上绝大多数的地方其精度可以达到1-50ms。而且可以使用加密确认的方式来防止病毒的协议攻击。
<!-- more -->

## 服务器配置

### 安装NTP服务

```bash
yum install -y ntp
```

### 修改ntp配置文件

查找最近的时间同步服务器，网址在<http://www.pool.ntp.org/zone/asia>

编辑 /etc/ntp.conf，增加时间同步服务器：

```
server 0.asia.pool.ntp.org
server 1.asia.pool.ntp.org
server 2.asia.pool.ntp.org
server 3.asia.pool.ntp.org
server 127.127.1.0 iburst    # 当外部时间服务器不可用，使用本机时间作为时间服务的标准
fudge 127.127.1.0 stratum 10 #这个值不能太高0-15，太高会报错
```

添加允许访问的ip段

```bash
restrict 192.168.217.0 mask 255.255.255.0 nomodify notrap
restrict 192.168.212.0 mask 255.255.255.0 nomodify notrap
```

### 配置防火墙

开启防火墙ntp默认端口udp123

```bash
firewall-cmd --permanent --zone=public --add-port=123/udp
firewall-cmd --reload
```

### 开机启动服务

```bash
systemctl enable ntpd
systemctl start ntpd
systemctl status ntpd
```

### 验证NTP服务

```bash
ntpq -p
date -R
```

## 客户端配置

安装的NTP跟上面的步骤一样

### 修改配置

将上面的NTP服务器作为客户端同步NTP时间服务器，`vim /etc/ntp.conf`

```
#配置允许NTP Server时间服务器主动修改本机的时间
restrict 192.168.217.161 nomodify notrap noquery
#注释掉其他时间服务器
#server 0.centos.pool.ntp.org iburst
#server 1.centos.pool.ntp.org iburst
#server 2.centos.pool.ntp.org iburst
#server 3.centos.pool.ntp.org iburst
#配置时间服务器为本地搭建的NTP Server服务器
server 192.168.217.161
```

### 与NTP server服务器同步一下时间

```bash
ntpdate -u 192.168.217.161
```

### 查看ntp同步状态

能看到已经成功同步

```bash
ntpq -p
date -R
```

## 修正时区

最后讲一下怎样修改linux的时区

```bash
rm -rf /etc/localtime    #删除当前默认时区
ln -s /usr/share/zoneinfo/Asia/Shanghai /etc/localtime 
```

将时区修改为中国上海的时区，当然，也可以设置中国香港或北京的时间。
