---
title: 树莓派4B安装Ubuntu Server 22.0.4
date: 2023-06-01 12:23:33 +0800
toc: true
categories: [ linux ]
tags: [ linux ]
---

介绍下树莓派没办法连接显示器和键盘的情况下怎样安装Ubuntu Server 22, 以及如何确定树莓派的 IP 地址并登陆进去等一些实用小技巧. 聪明的你还可以用这些学到的新技巧去扫描你的网络。

准备软件：

* [SDFormatter 5.0.2](https://pan.baidu.com/s/1fpKnVL8Kcpd4OidMMUhkKw?pwd=3zh5) 用来格式化闪存卡
* [Raspberry Pi Imager](https://downloads.raspberrypi.org/imager/imager_latest.exe) 用来烧录镜像
* [Ubuntu Server 22.0.4](https://old-releases.ubuntu.com/releases/22.04/ubuntu-22.04.1-preinstalled-server-arm64+raspi.img.xz) 操作系统镜像文件

<!-- more -->

## 格式化闪存卡
把闪存卡插入到读卡器中，插入笔记本/电脑的USB插口中，然后打开SDFormatter直接格式化即可。

## 烧录镜像
打开Raspberry Pi Imager，选择自定义的镜像并选中上面已经下载好的Ubuntu镜像，然后再选择存储卡，点击烧录即可。

## 配置网络
烧录完了先不要急着拔掉闪存卡，这里可以先配置下网络，后面可以SSH连接上，这样就不需要键盘鼠标和显示器了。在闪存卡的根目录找到network-config文件，用记事本打开。将下面的这段前面注释解开：
```yaml
wifis:
  wlan0:
    dhcp4: true
    optional: true
    access-points:
      "你的无线SSID":
        password: "你的无线密码"
```
注意无线网络名和密码都是英文双引号括起来，并且这是个yaml文件格式，缩进一定要看清楚。
配置完毕后, 将 SD 卡插入树莓派, 第一次启动后配置还没有生效, 需要重启一遍, 第二次应该就能自动连接 WiFi 了


## 树莓派通电

先安装上散热片，时候注意不要手抖，以免歪斜，贴纸双面，注意轻撕。
![img.png](https://xnstatic-1253397658.file.myqcloud.com/20230729-05.png)

1. 通电之前推荐先扫描一下局域网ip，这样好判断树莓派ip是哪一个。
2. 在电脑上使用“弹出”方式，弹出内存卡。直接将内存卡取出，有可能会损坏内存卡上的文件，推荐优雅取出。
3. 将烧录好的闪存卡插入树莓派，然后连接电源通电。

把 SD 卡插入树莓派, 然后给树莓派插上USB-C供电, 确认红灯亮起 (供电正常), 
然后绿灯开始闪烁 (机器运行)，树莓派就开始正常工作了。

## 查找IP地址
下载安装nmap之后，只需要输入如下命令就可以扫描局域网内的在线主机。
```
nmap -sP 192.168.1.0/24
```
或者如果不想下载nmap，也可以使用系统自带的工具。打开电脑的cmd命令行界面执行如下命令：
```
for /L %i IN (1,1,254) DO ping -w 1 -n 1 192.168.1.%i
arp -a
```

## SSH连接
用你的 SSH 客户端去连接刚才确定的树莓派的 IP 地址, 就可以连接上树莓派了.

默认的用户名和密码都是 "ubuntu", 第一次登陆后系统会要求你修改密码. 安全起见, 一定要修改密码.

## 无线配置 Debug
如果错过了在第一次启动前配置 WiFi, 或者配置的无线有问题, 那么可以按照如下方法 Debug:

首先看一下无线网卡的名称:
```
root@ubuntu:/home/ubuntu# ls /sys/class/net
eth0  lo  wlan0
```

一般情况有线网卡都叫 "eth", 无线网卡都叫 "wlan" (不过也有很多特例...)

我们可以看到 无线网卡叫 "wlan0"。然后编辑 /etc/netplan/50-cloud-init.yaml 文件
然后将之前的 WiFi 配置添加到配置文件最后:
```yaml
# This file is generated from information provided by the datasource. Changes
# to it will not persist across an instance reboot. To disable cloud-init's
# network configuration capabilities, write a file
# /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg with the following:
# network: {config: disabled}
network:
    ethernets:
        eth0:
            dhcp4: true
            optional: true
    version: 2
    wifis:
        wlan0:
            dhcp4: true
            optional: true
            access-points:
                "你的无线SSID":
                    password: "你的无线密码"
```
然后运行:
```bash
sudo netplan generate
sudo netplan apply
```

按道理这样就应该能让WiFi正常连接了. 仍然可以用 ip a 命令查看 wlan0 是不是自动获得了 IP 地址. 
获得了 IP 地址即代表连接到了目标网络并且 DHCP 服务器工作正常, 给你自动分配了地址.

