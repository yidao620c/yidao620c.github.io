---
layout: post
title: CentOS7通过samba共享文件夹
date: 2018-02-20 19:09:22 +0800
toc: true
categories: linux
tags:
  - samba
abbrlink: 22054
---

Samba是在Linux系统上实现SMB协议的一个免费软件，由服务器及客户端程序构成。
SMB(Server Messages Block, 信息服务块)是一种在局域网上共享文件和打印机的一种通信协议，
它为局域网内的不同计算机之间提供文件及打印机等资源的共享服务。
SMB协议是客户机/服务器型协议，客户机通过该协议可以访问服务器上的共享文件系统，打印机及其他资源。 

比如我想共享/home/samba这个文件夹给其他计算机使用。

## 安装samba

```
yum install -y samba
```

## 创建samba用户

```
useradd samba
```

## 修改samba配置

配置文件是/etc/samba/smb.conf

```
# See smb.conf.example for a more detailed config file or
# read the smb.conf manpage.
# Run 'testparm' to verify the config is correct after
# you modified it.

[global]
    workgroup = SAMBA   #samba的工作组，设置成 Windows 的工作组
    security = user     #安全选项，可以是 share|user|server|domain，安全级别递增
    passdb backend = tdbsam
    printing = cups
    printcap name = cups
    load printers = yes
    cups options = raw

[homes]   #共享默认会将用户的主目录共享 , 这是不安全的 , 可以将其注释
    comment = Home Directories
    valid users = %S, %D%w%S
    browseable = No
    read only = No
    inherit acls = Yes

[printers]   #打印机共享
    comment = All Printers
    path = /var/tmp
    printable = Yes
    create mask = 0600
    browseable = No

[print$]
    comment = Printer Drivers
    path = /var/lib/samba/drivers
    write list = root
    create mask = 0664
    directory mask = 0775

[rootdir]   #自定义的共享文件夹
    comment = SambaRoot
    path = /home/samba/   #共享的路径
    read only = No
```

注意，自己修改时去掉 `#` 后面的备注

## 添加 Samba 用户

添加刚刚创建的samba用户，根据提示设置相应的密码

```
smbpasswd -a samba
```

smbpasswd 命令是用于维护 Samba 服务器的用户帐号的，具体如下：

```
// 添加 Samba 用户帐号
# smbpasswd -a sambauser 
// 禁用 Samba 用户帐号
# smbpasswd -d sambauser
// 启用 Samba 用户帐号
# smbpasswd -e sambauser
// 删除 Samba 用户帐号
# smbpasswd -x sambauser
```

## 启动 Samba 服务

启动、停止、查看相关命令

```
systemctl start smb
systemctl stop smb
systemctl status smb
```

##  Windows 访问共享目录

直接 Win + R , 在运行界面输入 \\192.168.1.20， 也就是你的 Linux 主机地址，会弹出用户名密码输入界面，
输入刚刚设置的用户名密码就可以访问。

## 常见问题

如果 Windows 下访问 Linux 下共享目录 , 提示没有权限

1. 确保 Linux 下防火墙关闭或者是开放共享目录权限
1. 确保 Samba 服务器配置文件 smb.conf 设置没有问题
1. 确保 setlinux 关闭 , 可以用 `setenforce 0` 命令执行; 默认 SELinux 禁止网络上对 Samba 服务器上的共享目录进行写操作

Samb 还需要开放下面四个端口

```
UDP 137、UDP 138、TCP 139、TCP 445
```
