---
title: Docker每天学一点11 - 容器网络macvlan
date: 2019-03-16 15:22:38
toc: true
categories: [ k8s ]
tags: [ docker ]
---

除了 overlay，docker 还开发了另一个支持跨主机容器网络的 driver：macvlan。

macvlan 本身是 linxu kernel 模块，其功能是允许在同一个物理网卡上配置多个 MAC 地址，
即多个 interface，每个 interface 可以配置自己的 IP。macvlan 本质上是一种网卡虚拟化技术，Docker 用 macvlan 实现容器网络就不奇怪了。

macvlan 的最大优点是性能极好，相比其他实现，macvlan 不需要创建 Linux bridge，
而是直接通过以太 interface 连接到物理网络。下面我们就来创建一个 macvlan 网络。
<!-- more -->

## 环境准备

我们会使用 host1 和 host2 上单独的网卡 enp0s3 创建 macvlan。为保证多个 MAC 地址的网络包都可以从 enp0s3 通过，我们需要打开网卡的混杂模式。

```
ip link set enp0s3 promisc on
```

因为我的环境上面 host1 和 host2 是 VirtualBox 虚拟机，还需要在网卡配置选项页中设置混杂模式。这个在前面overlay网络互通时候已经设置过。

## 创建 macvlan 网络

先看网卡配置：

```
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 08:00:27:78:b4:16 brd ff:ff:ff:ff:ff:ff
    inet 192.168.1.21/24 brd 192.168.1.255 scope global noprefixroute enp0s3
       valid_lft forever preferred_lft forever
    inet6 fe80::a00:27ff:fe78:b416/64 scope link 
       valid_lft forever preferred_lft forever
```

ip地址为192.168.1.21，为了不和这个IP冲突，我们需要在创建macvlan的时候，排除此IP，使用docker network命令创建：

```
docker network create -d macvlan \
 --subnet=192.168.1.0/24 \
 --gateway=192.168.1.1 \
 --aux-address="exclude_host=192.168.1.21" \
 -o parent=enp0s3 mac_net1
```

1. `-d macvlan` 指定 driver 为 macvlan。
1. macvlan 网络是 local 网络，为了保证跨主机能够通信，用户需要自己管理 IP subnet。
1. 与其他网络不同，docker 不会为 macvlan 创建网关，这里的网关应该是真实存在的，否则容器无法路由。
1. -o parent 指定使用的网络 interface。

在 host2 中也要执行相同的命令创建macvlan网络。

在 host1 中运行容器 bbox1 并连接到 mac_net1，指定ip地址为192.168.1.10

```
docker run -itd --name bbox1 --ip=192.168.1.10 --network mac_net1 busybox
```

在 host2 中运行容器 bbox2 并连接到 mac_net1，指定ip地址为192.168.1.11

```
docker run -itd --name bbox2 --ip=192.168.1.11 --network mac_net1 busybox
```

验证 bbox1 和 bbox1 的连通性

```
[root@host2 ~]# docker exec bbox2 ping 192.168.1.10
PING 192.168.1.10 (192.168.1.10): 56 data bytes
64 bytes from 192.168.1.10: seq=0 ttl=64 time=4.775 ms
64 bytes from 192.168.1.10: seq=1 ttl=64 time=0.616 ms
```

ping主机名：

```
[root@host2 ~]# docker exec bbox2 ping -c 2 bbox1
ping: bad address 'bbox1'
```

可见 docker 没有为 macvlan 提供 DNS 服务，这点与 overlay 网络是不同的。
    
