---
title: netcat网络工具简介
date: 2020-06-03 03:15:15 +0800
toc: true
categories: [ 开发工具 ]
tags: [ 网络工具 ]
---

netcat 是 Linux 系统中的网络工具，其通过 TCP 和 UDP 协议在网络中读写数据。如果与其他工具结合，
以及加上重定向功能，还可以实现很多不同的功能。所以其以体积小功能灵活而著称，可以用来做很多网络相关的工作。

CentOS7中安装命令
```bash
yum install nmap-ncat
```

<!-- more -->

## 常用参数
netcat 使用的基本形式为：`nc 参数 目的地址 端口`。

常用的参数说明如下：
```
-k  在当前连接结束后保持继续监听
-l  用作端口监听，而不是发送数据
-n  不使用 DNS 解析
-N  在遇到 EOF 时关闭网络连接
-p  指定源端口
-u  使用 UDP 协议传输
-v  (Verbose)显示更多的详细信息
-w  指定连接超时时间
-z  不发送数据
```
## 常用功能说明

**端口测试**

netcat 可以替代 telnet 工具来测试远程注意端口是否打开，如命令：
```
telnet 192.168.1.8 8080
```

可以用 nc 来代替，还可以同时指定多个端口或者一个端口范围：
```
nc -vz 192.168.1.8 8080
nc -v -z -w 3 192.168.56.1 22 80 8000 8080
nc -v -z -w 3 192.168.1.8 8080-8088
```

**连接测试**

如果在主机上配置了防火墙，想要测试一下开放的端口是否可以与外界联通，可以用 netcat 监听该端口，然后从外界尝试连接。

在主机上执行：
```
nc -l -p 8080
```

在其它主机上可以尝试连接以上主机打开的端口：
```
nc 192.168.1.8 8080
```

如果两台主机能够正常连通的话，就如同打开了一个聊天室，可以相互发送数据并显示。

**测试 UDP**

netcat 工具可以指定使用 UDP 协议传输数据，只需家还是那个 -u 参数即可。
如下则是使用 UDP 方式监听 8080 端口：
```
nc -u -l -p 8080
```

连接主机时也可以指定 UDP 方式：
```
nc -u 192.168.1.8 8080
```

**文件传输**

两台主机间想要传输文件，而又不能使用 scp/szrz 等工具时，则可以用 netcat 代替。在需要接受文件的主机上执行：
```
nc -l -p 8080 > test.txt
```

然后再另一台主机上向其传输文件：
```
nc 192.168.1.8 8080 < test.txt
```

如何对传输的数据不放心，还可以进行加密，如使用 mcrypt 工具加密数据：
```bash
# 服务端，使用 mcrypt 工具加密数据，需要输入密码
nc localhost 1567 | mcrypt –flush –bare -F -q -d -m ecb > file.txt

# 客户端，使用 mcrypt 工具解密数据，需要输入密码
mcrypt –flush –bare -F -q -m ecb < file.txt | nc -l 1567
```