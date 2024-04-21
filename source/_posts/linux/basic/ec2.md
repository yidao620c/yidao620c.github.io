---
title: EC2虚拟主机搭建SS
date: 2017-09-30 17:29:33
toc: true
categories: [ linux ]
tags: [ linux ]
---

作为一个离开 Google 生活就无法自理的人类，我曾经发帖、提问、翻遍各种网站，四处寻找靠谱的科学上网利器。
网上也买过好多，自己也用过一些开源免费的proxy，最后都会出现各种莫名其妙的问题，各种不稳定。

最后我选择自己动手，丰衣足食，利用EC2虚拟主机搭建SS，这个东东是由 [Clowwindy](https://github.com/Clowwindy) 开发的一款软件，
其作用本来是加密传输资料。当然，也正因为它加密传输资料的特性，使得XXX没法将由它传输的资料和其他普通资料区分开来，
也就不能干扰我们访问那些「不存在」的网站了。

如果你希望不花钱就能用上优质的服务──醒醒，别做梦了，免费和优质从来不可能划上等号。
不过想要共享或建立多账户来出售的话，能赚钱也说不定🙃
<!-- more -->

## 创建EC2虚拟主机

先去亚马逊AWS上面注册一个账号：<https://amazonaws-china.com/cn/>

登录后进入EC2的控制台，然后在右上角区域里面切换至东京，
选择左边的"实例" ——> 启动实例 ——> AWS Marketplace ——> 搜索"centos7"，目前最新的版本是CentOS7.4

![](https://xnstatic-1253397658.file.myqcloud.com/ss01.png)

然后点击"continue"，默认选中符合条件的免费套餐。

![](https://xnstatic-1253397658.file.myqcloud.com/ss02.png)

然后点击"审核和启动"，这里编辑一下安全组信息，创建一个新安全组，类型里面选择"所有流量"即可。

启动之后，回到实例的界面，然后点击"连接"，复制上面的实例名

![](https://xnstatic-1253397658.file.myqcloud.com/ss03.png)

然后用xshell工具来连接，主机名选择上面实例详情的名称，使用密钥对来登录，用户名选择centos即可。

## 部署SS

好了，现在开始正式部署ss了，这里使用 [teddysun](https://teddysun.com/342.html) 的一键安装脚本。

先切换到root用户，可使用 `sudo passwd root` 先修改root密码，然后执行：

```bash
wget --no-check-certificate https://raw.githubusercontent.com/teddysun/shadowsocks_install/master/shadowsocks.sh
chmod +x shadowsocks.sh
./shadowsocks.sh 2>&1 | tee shadowsocks.log
```

按照提示来，最后成功界面如下：

![](https://xnstatic-1253397658.file.myqcloud.com/ss05.png)

请把这些信息复制下来。

## TCP Fast Open

实际上只要具备上述四个信息，你就可以在自己的任意设备上进行登录使用了。但是为了更好的连接速度，你还需要多做几步。

首先是打开 TCP Fast Open，`vi /etc/rc.local` ，在最后增加如下内容：

```
echo 3 > /proc/sys/net/ipv4/tcp_fastopen
```

然后修改`/etc/sysctl.conf`，在最后增加如下内容：

```
net.ipv4.tcp_fastopen = 3
```

再打开一个 Shadowsocks 配置文件，编辑`/etc/shadowsocks.json`，修改如下：

```
"fast_open":true
```

最后，输入以下命令重启 Shadowsocks：

```
/etc/init.d/shadowsocks restart
```

## 安装SS客户端

相比服务器端的安装，客户端的安装就简单了许多。首先，根据操作系统下载相应的客户端。

[Win 版客户端下载](https://github.com/shadowsocks/shadowsocks-windows/releases)

打开客户端，在「服务器设定」里新增服务器。然后依次填入服务器 IP、服务器端口、你设的密码和加密方式。

![](https://xnstatic-1253397658.file.myqcloud.com/ss06.png)

然后点击"启用系统代理"，选择PAC模式，在PAC中选择从xxx更新本地PAC，就可以实现科学上网了。

## 开启BBR加速

可以起到单边加速TCP连接的效果。

Google提交到Linux主线并发表在ACM queue期刊上的TCP-BBR拥塞控制算法。继承了Google"先在生产环境上部署，再开源和发论文"的研究传统。
TCP-BBR已经再YouTube服务器和Google跨数据中心的内部广域网(B4)上部署。由此可见出该算法的前途。

TCP-BBR的目标就是最大化利用网络上瓶颈链路的带宽。一条网络链路就像一条水管，要想最大化利用这条水管，最好的办法就是给这跟水管灌满水。

BBR解决了两个问题：

1. 在有一定丢包率的网络链路上充分利用带宽。非常适合高延迟，高带宽的网络链路。
2. 降低网络链路上的buffer占用率，从而降低延迟。非常适合慢速接入网络的用户。
   Google 在 2016年9月份开源了他们的优化网络拥堵算法BBR，最新版本的 Linux内核(4.9-rc8)中已经集成了该算法。

对于TCP单边加速，并非所有人都很熟悉，不过有另外一个大名鼎鼎的商业软件"锐速"，相信很多人都清楚。
特别是对于使用国外服务器或者VPS的人来说，效果更佳。

BBR项目地址：

```
https://github.com/google/bbr
```

升级内核，第一步首先是升级内核到支持BBR的版本：

1.yum更新系统版本：

```
yum update
```

2.查看系统版本：

```
[centos@ip-172-31-23-1 ~]$ cat /etc/redhat-release 
CentOS Linux release 7.5.1804 (Core)
```

3.安装elrepo并升级内核：

```
[root@ip-172-31-23-1 ~]# rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org
[root@ip-172-31-23-1 ~]# rpm -Uvh http://www.elrepo.org/elrepo-release-7.0-2.el7.elrepo.noarch.rpm
[root@ip-172-31-23-1 ~]# yum --enablerepo=elrepo-kernel install kernel-ml -y
```

4.更新grub文件并重启系统：

```
[root@ip-172-31-23-1 ~]# egrep ^menuentry /etc/grub2.cfg | cut -f 2 -d \'
CentOS Linux (4.20.0-1.el7.elrepo.x86_64) 7 (Core)
CentOS Linux (3.10.0-862.3.2.el7.x86_64) 7 (Core)
CentOS Linux (0-rescue-b30d0f2110ac3807b210c19ede3ce88f) 7 (Core)
[root@ip-172-31-23-1 ~]# grub2-set-default 0
[root@ip-172-31-23-1 ~]# reboot
```

5.重启完成后查看内核是否已更换为4.xx版本：

```
[centos@ip-172-31-23-1 ~]$ uname -r
4.20.0-1.el7.elrepo.x86_64
```

6.开启bbr：

```
vim /etc/sysctl.conf    # 在文件末尾添加如下内容
net.core.default_qdisc = fq
net.ipv4.tcp_congestion_control = bbr
```

7.加载系统参数：

```
[root@ip-172-31-23-1 ~]# sysctl -p
net.core.default_qdisc = fq
net.ipv4.tcp_congestion_control = bbr
```

输出了我们添加的那两行配置代表正常。

8.确定bbr已经成功开启：

```
[root@ip-172-31-23-1 ~]#  sysctl net.ipv4.tcp_available_congestion_control
net.ipv4.tcp_available_congestion_control = reno cubic bbr
[root@ip-172-31-23-1 ~]# lsmod | grep bbr
tcp_bbr                20480  5
```

输出内容如上，则表示bbr已经成功开启。

---------------

参考文章

* [科学上网的终极姿势](https://zoomyale.com/2016/vultr_and_ss/)
* [CentOS7.4搭建shadowsocks，以及配置BBR加速](http://blog.51cto.com/zero01/2064660)

**备注**

2018年12月更新，由于我的私钥弄丢了，登录不上EC2主机，所以重新创建了一个实例，选了CentOS7.5，并使用了BBR加速。
所以创建EC2的时候私钥千万一定要自己保存好啊。

