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
烧录完了先不要急着拔掉闪存卡，这里可以先配置下网络，后面可以SSH连接上，这样就不需要键盘鼠标和显示器了。在闪存卡的根目录找到network-config文件，
用记事本打开。将下面的这段前面注释解开：
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

## 配置静态IP
用户SSH连接上Ubuntu Server 22.0.4之后，第一件事就是将动态DHCP分配IP修改成静态IP地址。
```
sudo vim /etc/netplan/50-cloud-init.yaml
```
修改文件内容如下：
```yaml
network:
    ethernets:
        eth0:
            dhcp4: true
            optional: true
    version: 2
    wifis:
        wlan0:
            dhcp4: no
            optional: true
            addresses: [192.168.1.7/24]
            routes:
                - to: default
                  via: 192.168.1.1
            nameservers:
                addresses: [192.168.1.1]
            access-points:
                1209-5G:
                    password: '12091209'
```
然后执行
```
sudo netplan generate # 没有报错则ok
sudo netplan apply # 此时应用静态ip修改，IP地址发生改变
```

## 树莓派备份

* https://github.com/mghcool/Raspberry-backup
* https://github.com/nanhantianyi/rpi-backup

优选第一个方案。

## Ubuntu系统关闭unattended upgrades
在使用ubuntu虚拟机的过程中，遇到关机或重启很慢的问题，提示有一个UU（unattended upgrades）进程在工作，需要等待30min。

经过一番了解，发现这个UU（unattended upgrades）进程就是ubuntu搞的一个类似于windows系统的自动更新程序，
目的是让普通用户的系统能随时保持最新，但对于开发来说实属麻烦。这个进程会在后台自动下载和安装系统更新文件，会阻止关机，有时候还会阻止你安装其他软件

```
sudo apt remove unattended-upgrades
```
从此再也不受这个问题困扰了，搞定！重启系统试试吧！
这个方法完全关闭自动更新了，但如果还想手动更新怎么办？完全不用担心，因为还有如下两条指令，在需要的时候手动输入更新就好了
```
sudo apt update
sudo apt upgrade
```
