---
title: CentOS7忘记root密码解决办法
date: '2018-02-03 09:06:17 +0800'
toc: true
categories: [ linux ]
tags: [ linux ]
---

平日里让运维人员头疼的事情已经很多了，因此偶尔把Linux系统的密码忘记了并不用慌，
只需简单几步就可以完成密码的重置工作。本文基于CentOS7环境进行操作，由于CentOS的版本是有差异的，
继续之前请确定好版本。
<!--more-->

第1步：重启Linux系统主机并出现引导界面时，按下键盘上的e键进入内核编辑界面，如图所示。

![](https://xnstatic-1253397658.file.myqcloud.com/linux082601.png)

第2步：在linux16参数这行的最后面追加"rd.break"参数，然后按下Ctrl + X组合键来运行修改过的内核程序。
![](https://xnstatic-1253397658.file.myqcloud.com/linux082602.png)

第3步：进入到系统的紧急求援模式，如图所示。
![](https://xnstatic-1253397658.file.myqcloud.com/linux082603.png)

第4步：依次输入以下命令，等待系统重启操作完毕，耐心等待后就可以使用新密码来登录Linux系统了。

```bash
mount -o remount,rw /sysroot
chroot /sysroot
passwd
touch /.autorelabel
exit
reboot
```

