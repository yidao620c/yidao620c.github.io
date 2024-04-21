---
title: 使用VirtualBox 6安装Centos7
date: '2018-02-01 00:29:12 +0800'
toc: true
categories: [ linux ]
tags: [ linux ]
---

VirtualBox 是一款开源虚拟机软件。VirtualBox 是由德国 Innotek 公司开发，
由Sun Microsystems公司出品的软件，使用Qt编写，在 Sun 被 Oracle 收购后正式更名成
Oracle VM VirtualBox。

本文演示如何在Win10上面安装VirtualBox 6，并安装CentOS 7操作系统。
<!--more-->

## 安装和配置

首先下载VirtualBox 6的安装文件并点击安装，这个不用多讲。下载地址：<https://www.virtualbox.org/wiki/Downloads>

然后下载centos7的mini.iso文件，下载地址：<https://www.centos.org/download/>

打开VirtualBox 6，新建一个虚拟机，网络设置成桥接模式，并且在高级中设置网卡混杂模式为全部允许。
挂载刚刚下载的镜像文件，然后一步步去安装，最好选择语言为English界面。

安装完成后配置静态IP地址，先用ifconfig看看windows宿主机上面的ip。
然后编辑文件`/etc/sysconfig/network-scripts/ifcfg-enp0s3`，注意IP设置为跟宿主机上同网段。

```
TYPE=Ethernet
BOOTPROTO=static
DEFROUTE=yes
PEERDNS=yes
PEERROUTES=yes
IPV4_FAILURE_FATAL=no
IPV6_FAILURE_FATAL=no
NAME=enp0s3
DEVICE=enp0s3
ONBOOT=yes
IPADDR=192.168.1.20
NETMASK=255.255.255.0
GATEWAY=192.168.1.1
DNS1=114.114.114.114
```

SSH连接报一个警告：`Reject X11 forwarding...`。

```bash
yum -y install xorg-x11-xauth
```

然后再修改`/etc/ssh/sshd_config`

```
X11Forwarding yes
AllowAgentForwarding yes
```

想修改为多用户状态只需执行：

```bash
systemctl set-default multi-user.target
```

修改为图形界面执行

```bash
systemctl set-default graphical.target
```

```bash
yum update
yum -y install net-tools wget telnet vim
```

## 后台运行

关闭虚拟机。在win10上面创建一个bat脚本，以后台方式启动虚拟机：

```
@echo off
cd /d "D:\Program Files\Oracle\VirtualBox"
VBoxManage.exe startvm "master" --type headless
VBoxManage.exe startvm "host1" --type headless
VBoxManage.exe startvm "host2" --type headless
```

## VirtualBox常用命令

```
#查看帮助
VBoxManage help

#查看有哪些虚拟机
VBoxManage list vms
 
#查看运行着的虚拟机
VBoxManage list runningvms
 
#查看虚拟的详细信息
VBoxManage showvminfo <vm_name>

#开启虚拟机在后台运行
VBoxManage startvm <vm_name> --type headless
 
#关闭虚拟机
VBoxManage controlvm <vm_name> acpipowerbutton
 
#强制关闭虚拟机
VBoxManage controlvm <vm_name> poweroff

#删除虚拟机
VBoxManage unregistervm  <vm_name> --delete

#修改虚拟机名
VBoxManage modifyvm <vm_name> --name <new_vm_name>
```

## 远程桌面设置

```bash
VBoxManage controlvm <vm_name> vrdeport 3399 
VBoxManage controlvm <vm_name> vrde on 
```

然后在windows下面用mstsc命令打开远程桌面，输入`IP:3399`就可以进入虚拟机桌面了

