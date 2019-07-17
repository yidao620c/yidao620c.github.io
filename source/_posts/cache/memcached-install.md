---
layout: post
title: linux上安装memcached
date: 2015-04-16 15:52:35 +0800
toc: true
categories: cache
tags: memcached
abbrlink: 7044
---

下载memcached与libevent的安装文件：

memcached下载地址：[memcached-1.4.15.tar.gz](http://memcached.googlecode.com/files/memcached-1.4.15.tar.gz)

libevent下载地址：[libevent-2.0.21-stable.tar.gz](https://github.com/downloads/libevent/libevent/libevent-2.0.21-stable.tar.gz)


### 具体安装步骤

#### 安装libevent

1. 由于memcached依赖于libevent，因此需要安装libevent。由于linux系统可能默认已经安装libevent，执行命令：
``` bash
rpm -qa|grep libevent 
```
1. 查看系统是否带有该安装软件，如果有执行命令:
``` bash
# 由于系统自带的版本旧，忽略依赖删除
rpm -e libevent-1.4.13-4.el6.x86_64 –nodeps
```
1. 安装libevent命令：
``` bash
tar zxvf libevent-2.0.21-stable.tar.gz
cd libevent-2.0.21-stable
./configure --prefix=/usr/local/libevent
make
make install
```
至此libevent安装完毕；

#### 安装memcached

``` bash
tar zxvf memcached-1.4.2.tar.gz
cd memcached-memcached-1.4.2
./configure --prefix=/usr/local/memcached --with-libevent=
/usr/local/libevent/
make
make install
```
至此memcached安装完毕；

#### 可能存在的错误以及解决方案

如果出现客户端连接不上memcached的情况，请将防火墙关闭或将防火墙中的memcached端口（11211端口）打开。

#### 启动memcached

打开一个终端，输入以下命令：
``` bash
/usr/local/memcached/bin/memcached -d -m 256 -u root -p 11211 -c 1024 –P /tmp/memcached.pid
```

启动参数说明：
``` none
-d 选项是启动一个守护进程。
-u root 表示启动memcached的用户为root。
-m 是分配给Memcache使用的内存数量，单位是MB，默认64MB。
-M return error on memory exhausted (rather than removing items)。
-u 是运行Memcache的用户，如果当前为root 的话，需要使用此参数指定用户。
-p 是设置Memcache的TCP监听的端口，最好是1024以上的端口。
-c 选项是最大运行的并发连接数，默认是1024。
-P 是设置保存Memcache的pid文件。
```

另外还有个更详细的参数说明：
``` none
memcached 1.4.2
-p <num监听的TCP端口(默认: 11211)
-U <num监听的UDP端口(默认: 11211, 0表示不监听)
-s <file用于监听的UNIX套接字路径（禁用网络支持）
-a <maskUNIX套接字访问掩码，八进制数字（默认：0700）
-l <ip_addr监听的IP地址。（默认：INADDR_ANY，所有地址）
-d 作为守护进程来运行。
-r 最大核心文件限制。
-u <username设定进程所属用户。（只有root用户可以使用这个参数）
-m <num单个数据项的最大可用内存，以MB为单位。（默认：64MB）
-M 内存用光时报错。（不会删除数据）
-c <num最大并发连接数。（默认：1024）
-k 锁定所有内存页。注意你可以锁定的内存上限。
试图分配更多内存会失败的，所以留意启动守护进程时所用的用户可分配的内存上限。
（不是前面的 -u <username参数；在sh下，使用命令"ulimit -S -l NUM_KB"来设置。）
-v 提示信息（在事件循环中打印错误/警告信息。）
-vv 详细信息（还打印客户端命令/响应）
-vvv 超详细信息（还打印内部状态的变化）
-h 打印这个帮助信息并退出。
-i 打印memcached和libevent的许可。
-P <file保存进程ID到指定文件，只有在使用 -d 选项的时候才有意义。
-f <factor块大小增长因子。（默认：1.25）
-n <bytes分配给key+value+flags的最小空间（默认：48）
-L 尝试使用大内存页（如果可用的话）。提高内存页尺寸可以减少"页表缓冲（TLB）"丢失次数，提高运行效率。
为了从操作系统获得大内存页，memcached会把全部数据项分配到一个大区块。
-D <char使用 <char作为前缀和ID的分隔符。
这个用于按前缀获得状态报告。默认是":"（冒号）。
如果指定了这个参数，则状态收集会自动开启；如果没指定，则需要用命令"stats detail on"来开启。
-t <num使用的线程数（默认：4）
-R 每个连接可处理的最大请求数。
-C 禁用CAS。
-b 设置后台日志队列的长度（默认：1024）
-B 绑定协议 - 可能值：ascii,binary,auto（默认）
-I 重写每个数据页尺寸。调整数据项最大尺寸。
```

也可以启动多个守护进程，但是端口不能重复

#### 查看memcached启动命令：
``` bash
ps aux|grep memcached
```
#### 停止memcached

打开一个终端，输入以下命令：
``` bash
ps -ef | grep memcached或者上面的ps命令也行，第二个字段为PID，比如10068
```
输入一下命令终止memcached服务：
``` bash
kill -9 10068
```
