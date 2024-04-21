---
title: Docker每天学一点10 - 容器网络overlay
date: 2019-03-15 09:56:12
toc: true
categories: [ k8s ]
tags: [ docker ]
---

前面已经学习了 Docker 的几种网络方案：none、host、bridge 和 joined 容器，
它们解决了单个 Docker Host 内容器通信的问题。本章的重点则是讨论跨主机容器间通信的方案。

跨主机网络方案包括：

* docker 原生的 overlay 和 macvlan。
* 第三方方案：常用的包括 flannel、weave 和 calico。

docker 网络是一个非常活跃的技术领域，不断有新的方案开发出来，那么要问个非常重要的问题了：
如此众多的方案是如何与 docker 集成在一起的？

答案是：libnetwork 以及 CNM（草泥马？）。
<!-- more -->

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

一个容器一个 Sandbox，每个 Sandbox 都有一个 Endpoint 连接到 Network 1，第二个 Sandbox 还有一个 Endpoint 将其接入 Network

2.

libnetwork CNM 定义了 docker 容器的网络模型，按照该模型开发出的 driver 就能与 docker daemon 协同工作，实现容器网络。
docker 原生的 driver 包括 none、bridge、overlay 和 macvlan，第三方 driver 包括 flannel、weave、calico 等。

![](https://xnstatic-1253397658.file.myqcloud.com/docker-network22.webp)

接下来我们将详细讨论各种跨主机网络方案，首先学习 Overlay。

## 环境准备

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

## 创建overlay网络

在host1上面创建overlay网络ov_net1:

```
docker network create -d overlay ov_net1
```

`-d overlay` 指定 driver 为 overaly。

查看当前所有docker网络：

```
[root@localhost ~]# docker network ls
NETWORK ID          NAME                DRIVER              SCOPE
a72ce58f3bd2        bridge              bridge              local
bc794f4afe7a        host                host                local
3e0372add56d        none                null                local
4c21b88438fb        ov_net1             overlay             global
```

注意到 ov_net1 的 SCOPE 为 global，而其他网络为 local。在 host2 上查看存在的网络：

```
[root@localhost ~]# docker network ls
NETWORK ID          NAME                DRIVER              SCOPE
3f811e49c9f1        bridge              bridge              local
a16bad2e5ceb        host                host                local
8b8ed97c4bcc        none                null                local
4c21b88438fb        ov_net1             overlay             global
```

host2 上也能看到 ov_net1。这是因为创建 ov_net1 时 host1 将 overlay 网络信息存入了 consul，
host2 从 consul 读取到了新网络的数据。之后 ov_net1 的任何变化都会同步到 host1 和 host2。

`docker network inspect` 查看 ov_net1 的详细信息：

```
[
    {
        "Name": "ov_net1",
        省略...
        "IPAM": {
            "Driver": "default",
            "Options": {},
            "Config": [
                {
                    "Subnet": "10.0.0.0/24",
                    "Gateway": "10.0.0.1"
                }
            ]
        },
    }
]
```

IPAM 是指 IP Address Management，docker 自动为 ov_net1 分配的 IP 空间为 10.0.0.0/24。

## 在 overlay 中运行容器

运行一个 busybox 容器并连接到 ov_net1：

```
docker run -itd --name bbox1 --network ov_net1 busybox
```

查看容器的网络配置：

```
[root@localhost ~]# docker exec bbox1 ip r
default via 172.18.0.1 dev eth1 
10.0.0.0/24 dev eth0 scope link  src 10.0.0.2 
172.18.0.0/16 dev eth1 scope link  src 172.18.0.2 
```

bbox1 有两个网络接口 eth0 和 eth1。eth0 IP 为 10.0.0.2，连接的是 overlay 网络 ov_net1。eth1 IP 172.18.0.2，
容器的默认路由是走 eth1，eth1 是哪儿来的呢？

其实，docker 会创建一个 bridge 网络 "docker_gwbridge"，为所有连接到 overlay 网络的容器提供访问外网的能力

```
[root@localhost ~]# docker network ls
NETWORK ID          NAME                DRIVER              SCOPE
a72ce58f3bd2        bridge              bridge              local
375d48203637        docker_gwbridge     bridge              local
bc794f4afe7a        host                host                local
3e0372add56d        none                null                local
4c21b88438fb        ov_net1             overlay             global
```

从 `docker network inspect docker_gwbridge` 输出可确认 docker_gwbridge 的 IP 地址范围是 172.18.0.0/16，
当前连接的容器就是 bbox1（172.18.0.2）

```
"IPAM": {
    "Driver": "default",
    "Options": null,
    "Config": [
        {
            "Subnet": "172.18.0.0/16",
            "Gateway": "172.18.0.1"
        }
    ]
},
"Internal": false,
"Attachable": false,
"Ingress": false,
"ConfigFrom": {
    "Network": ""
},
"ConfigOnly": false,
"Containers": {
    "384ea73907b3c57ea3a7d66d7e131ce17457cd24e4485bb386f8d8b97562e5ce": {
        "Name": "gateway_742d88ece54f",
        "EndpointID": "d6069af3fa630d2d3e0f80ee71325aaabb061d9435df370dbff93cb6c92a1478",
        "MacAddress": "02:42:ac:12:00:02",
        "IPv4Address": "172.18.0.2/16",
        "IPv6Address": ""
    }
},
```

而且此网络的网关就是网桥 docker_gwbridge 的 IP 172.18.0.1

```
[root@localhost ~]# ifconfig docker_gwbridge
docker_gwbridge: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 172.18.0.1  netmask 255.255.0.0  broadcast 172.18.255.255
        inet6 fe80::42:3bff:fe06:a807  prefixlen 64  scopeid 0x20<link>
```

这样容器 bbox1 就可以通过 docker_gwbridge 访问外网。

如果外网要访问容器，可通过主机端口映射，比如：

```
docker run -p 80:80 -d --net ov_net1 --name web1 httpd
```

验证完外网的连通性，下一节验证 overlay 网络跨主机通信。

## overlay 网络跨主机通信

在 host2 中运行容器 bbox2：

```
docker run -itd --name bbox2 --network ov_net1 busybox
```

查看bbox2容器内网络配置

```
[root@localhost ~]# docker exec bbox2 ip r
default via 172.18.0.1 dev eth1 
10.0.0.0/24 dev eth0 scope link  src 10.0.0.3 
172.18.0.0/16 dev eth1 scope link  src 172.18.0.2 
```

bbox2 IP 为 10.0.0.3，可以直接 ping bbox1：

```
[root@host2 ~]# docker exec bbox2 ping -c 2 bbox1
PING bbox1 (10.0.0.2): 56 data bytes
--- bbox1 ping statistics ---
2 packets transmitted, 0 packets received, 100% packet loss
```

哦吼吼，ping不通

找解决方案如下：

第一步，所有主机名都修改一下，改成不同
第二步，关闭selinux，防火墙开通几个端口或者完全关闭

```
iptables -A INPUT -p tcp -m tcp --dport 7946 -j ACCEPT
iptables -A INPUT -p udp -m udp --dport 7946 -j ACCEPT
iptables -A INPUT -p tcp -m tcp --dport 4789 -j ACCEPT
iptables -A INPUT -p 50 -j ACCEPT

systemctl stop firewalld
```

第三步，继续查找原因。我的几个机器是VirtualBox安装的，
确保在virtualbox设置中打开了host1和host2所有虚拟网卡的混杂模式。

成功搞定！

```
[root@host2 ~]# docker exec bbox2 ping -c 2 bbox1
PING bbox1 (10.0.0.2): 56 data bytes
64 bytes from 10.0.0.2: seq=0 ttl=64 time=1003.159 ms
64 bytes from 10.0.0.2: seq=1 ttl=64 time=2.794 ms

--- bbox1 ping statistics ---
2 packets transmitted, 2 packets received, 0% packet loss
```

可见 overlay 网络中的容器可以直接通信，同时 docker 也实现了 DNS 服务。

总结下overlay 网络的具体实现：

docker 会为每个 overlay 网络创建一个独立的 network namespace，其中会有一个 linux bridge br0，
endpoint 还是由 veth pair 实现，一端连接到容器中（即 eth0），另一端连接到 namespace 的 br0 上。

br0 除了连接所有的 endpoint，还会连接一个 vxlan 设备，用于与其他 host 建立 vxlan tunnel。
容器之间的数据就是通过这个 tunnel 通信的。逻辑网络拓扑结构如图所示（截的别人的图）：

![](https://xnstatic-1253397658.file.myqcloud.com/docker20190215-01.jpg)

要查看 overlay 网络的 namespace 可以在 host1 和 host2 上执行 ip netns
（请确保在此之前执行过 `ln -s /var/run/docker/netns /var/run/netns`），可以看到两个 host 上有一个相同的 namespace

```
[root@host2 ~]# ip netns
d72b529b1df2 (id: 1)
1-4c21b88438 (id: 0)
```

这就是 ov_net1 的 namespace，查看 namespace 中的 br0 上的设备。

```
[root@host2 ~]# ip netns exec 1-4c21b88438 brctl show
bridge name	bridge id		STP enabled	interfaces
br0		8000.626988b3db42	no		    veth0
							            vxlan0
```

查看 vxlan0 设备的具体配置信息可知此 overlay 使用的 VNI（VxLAN ID）为 256。

```
[root@host2 ~]# ip netns exec 1-4c21b88438 ip -d l show vxlan0
6: vxlan0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue master br0 state UNKNOWN mode DEFAULT group default 
    link/ether da:53:ff:c8:09:98 brd ff:ff:ff:ff:ff:ff link-netnsid 0 promiscuity 1 
    vxlan id 256 srcport 0 0 dstport 4789 proxy l2miss l3miss ageing 300 noudpcsum noudp6zerocsumtx noudp6zerocsumrx
```

## overlay网络隔离

不同的 overlay 网络是相互隔离的。我在host1上面创建第二个 overlay 网络 ov_net2 并运行容器 bbox3。

```
docker network create -d overlay ov_net2
docker run -itd --name bbox3 --network ov_net2 busybox
```

查看bbox3容器的网络

```
[root@host1 ~]# docker exec bbox3 ip r
default via 172.18.0.1 dev eth1 
10.0.1.0/24 dev eth0 scope link  src 10.0.1.2 
172.18.0.0/16 dev eth1 scope link  src 172.18.0.3
```

ip地址为10.0.1.2，尝试ping一下bbox1的ip地址（10.0.0.2）：

```
[root@host1 ~]# docker exec bbox3 ping -c 2 10.0.0.2
PING 10.0.0.2 (10.0.0.2): 56 data bytes

--- 10.0.0.2 ping statistics ---
2 packets transmitted, 0 packets received, 100% packet loss
```

ping 失败，可见不同 overlay 网络之间是隔离的。即便是通过 docker_gwbridge 也不能通信。

```
[root@host1 ~]# docker exec bbox3 ping -c 2 172.18.0.2
PING 172.18.0.2 (172.18.0.2): 56 data bytes

--- 172.18.0.2 ping statistics ---
2 packets transmitted, 0 packets received, 100% packet loss
```

docker 默认为 overlay 网络分配 24 位掩码的子网（10.0.X.0/24），所有主机共享这个 subnet，
容器启动时会顺序从此空间分配 IP。当然我们也可以通过 --subnet 指定 IP 空间。

```
docker network create -d overlay --subnet 10.22.1.0/24 ov_net3
```

## 总结Overlay网络

* Overlay网络会自动创建一个docker_gwbridge网卡，作用是为Docker容器提供上外网需求
* Docker容器不能通过docker_gwbridge网卡互相通信，即使在同一台Docker主机, 同一个Overlay网络也不行
* Docker容器间通信只能通过overlay网络
* Overlay网络间是相互隔离的，通过VXLan隔离
* Docker容器网络的命名空间与Overlay网络的命名空间通过一对veth pair连接起来
* Overlay网络独立的命名空间里面会有一个网桥br0
* Docker容器的veth pair对端veth0与vxlan0设备通过br0这个Linux bridge桥接在一起， br0在同一宿主机上起到虚拟交换机的作用，
  如果目标地址在同一宿主机上，则直接通信，如果不在则通过设置vxlan0这个VXLan设备进行跨主机通信
* VXLan0设备会在创建时，由Docker daemon 为其分配vxlan隧道ID，起到网络隔离的作用
* Docker Host集群会通过key/value存储共享数据，在7496端口上，互相之间通过gossip协议学习各个宿主机上运行了那些容器。
  守护进程根据这些数据在vxlan0设备上生成静态MAC转发表
* 根据静态MAC转发表的设置，通过UDP端口4789，将流量转发到对端宿主机网卡上
* 根据流量包中的VxLan隧道ID，将流量转发到对端宿主机的overlay网络的网络命名空间中。
* 对端宿主机的overlay网络的网络命名空间中br0网桥，起到虚拟交换机的作用，将流量根据MAC地址转发到对应容器内部。


