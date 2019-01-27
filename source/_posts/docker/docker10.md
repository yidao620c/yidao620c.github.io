---
title: Docker每天学一点10 - 容器网络Overlay
date: 2017-04-15 09:56:12
toc: true
categories: docker
tags: docker
---

前面已经学习了 Docker 的几种网络方案：none、host、bridge 和 joined 容器，
它们解决了单个 Docker Host 内容器通信的问题。本章的重点则是讨论跨主机容器间通信的方案。

跨主机网络方案包括：

* docker 原生的 overlay 和 macvlan。
* 第三方方案：常用的包括 flannel、weave 和 calico。

docker 网络是一个非常活跃的技术领域，不断有新的方案开发出来，那么要问个非常重要的问题了：

如此众多的方案是如何与 docker 集成在一起的？

答案是：libnetwork 以及 CNM（草泥马？）。<!--more-->

## libnetwork & CNM

libnetwork 是 docker 容器网络库，最核心的内容是其定义的 Container Network Model (CNM)，
这个模型对容器网络进行了抽象，由以下三类组件组成：

*Sandbox*

Sandbox 是容器的网络栈，包含容器的 interface、路由表和 DNS 设置。 Linux Network Namespace 是 Sandbox 的标准实现。
Sandbox 可以包含来自不同 Network 的 Endpoint。

*Endpoint*

Endpoint 的作用是将 Sandbox 接入 Network。Endpoint 的典型实现是 veth pair，后面我们会举例。
一个 Endpoint 只能属于一个网络，也只能属于一个 Sandbox。

*Network*

Network 包含一组 Endpoint，同一 Network 的 Endpoint 可以直接通信。Network 的实现可以是 Linux Bridge、VLAN 等。

下面是 CNM 的示例：

![](https://xnstatic-1253397658.file.myqcloud.com/docker-network21.webp)

一个容器一个 Sandbox，每个 Sandbox 都有一个 Endpoint 连接到 Network 1，第二个 Sandbox 还有一个 Endpoint 将其接入 Network 2.

libnetwork CNM 定义了 docker 容器的网络模型，按照该模型开发出的 driver 就能与 docker daemon 协同工作，实现容器网络。
docker 原生的 driver 包括 none、bridge、overlay 和 macvlan，第三方 driver 包括 flannel、weave、calico 等。

![](https://xnstatic-1253397658.file.myqcloud.com/docker-network22.webp)

接下来我们将详细讨论各种跨主机网络方案，首先学习 Overlay。

## Overlay

为支持容器跨主机通信，Docker 提供了 overlay driver，使用户可以创建基于 VxLAN 的 overlay 网络。
Docerk overlay 网络需要一个 key-value 数据库用于保存网络状态信息，包括 Network、Endpoint、IP 等。
Consul、Etcd 和 ZooKeeper 都是 Docker 支持的 key-vlaue 软件，我们这里使用 Consul。

目前实验环境我个人两台机器：host1和host2。在另外一台机器master(192.168.1.20)上面安装Consul。

```
docker run -d -p 8500:8500 -h consul --name consul progrium/consul -server -bootstrap
```

然后访问http://192.168.1.20:8500：

![](https://xnstatic-1253397658.file.myqcloud.com/docker100.jpg)

接下来修改 host1 和 host2 的 docker daemon 的配置文件`/usr/lib/systemd/system/docker.service`，
我只列出ExecStart最后部分：
```
-H unix:///var/run/docker.sock \
-H tcp://0.0.0.0:2376 \
--cluster-store=consul://192.168.1.20:8500 \
--cluster-advertise=enp0s3:2376
```

* --cluster-store 指定 consul 的地址。
* --cluster-advertise 告知 consul 自己的连接地址。

重启 docker daemon。
```
systemctl daemon-reload
systemctl restart docker.service
```

再次刷新consul，查看KEY/VALUE，发现多了两个节点：

![](https://xnstatic-1253397658.file.myqcloud.com/docker101.jpg)

最后的实验环境如下：

![](https://xnstatic-1253397658.file.myqcloud.com/docker102.jpg)




