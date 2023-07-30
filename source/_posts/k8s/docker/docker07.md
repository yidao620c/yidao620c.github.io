---
title: Docker每天学一点07 - 单主机网络
date: 2019-03-10 12:17:22
toc: true
categories: [ k8s ]
tags: [ docker ]
---

这一篇学习容器之间、容器和外界之间怎样相互通信。Docker 网络从覆盖范围可分为单主机的容器网络和跨主机的容器网络，
本章重点讨论前一种。对于更为复杂的多主机容器网络，后面的文章会专门讲。
<!-- more -->

## 三个网络

Docker 安装时会自动在 host 上创建三个网络，我们可用` docker network ls` 命令查看：

```bash
[root@VM_22_2_centos docker]# docker network ls
NETWORK ID          NAME                DRIVER              SCOPE
6813d9c10747        bridge              bridge              local
307cfbfa0d42        host                host                local
efefda795912        none                null                local
```

**none 网络**

挂在这个网络下的容器除了 lo，没有其他任何网卡。容器创建时，可以通过 `--network=none` 指定使用 none 网络

```bash
[root@VM_22_2_centos docker]# docker run -it --network=none busybox
Unable to find image 'busybox:latest' locally
latest: Pulling from library/busybox
07a152489297: Pull complete 
Digest: sha256:8cdbc15e55361edcfd25acd47b00cb7dd890a9816bf4273ccbaf7e6aa41945de
Status: Downloaded newer image for busybox:latest
/ # ifconfig
lo        Link encap:Local Loopback  
          inet addr:127.0.0.1  Mask:255.0.0.0
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1 
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)
```

一些对安全性要求高并且不需要联网的应用可以使用 none 网络。

**host 网络**

连接到 host 网络的容器共享 Docker host 的网络栈，容器的网络配置与 host 完全一样。可以通过 `--network=host` 指定使用 host
网络。

```bash
[root@VM_22_2_centos docker]# docker run -it --network=host busybox
/ # ip l
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast qlen 1000
    link/ether 52:54:00:cf:74:9f brd ff:ff:ff:ff:ff:ff
3: docker0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue 
    link/ether 02:42:d5:1d:92:2c brd ff:ff:ff:ff:ff:ff
29: veth68c4395@if28: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue master docker0 
    link/ether ee:92:96:50:92:87 brd ff:ff:ff:ff:ff:ff
```

在容器中可以看到 host 的所有网卡，并且连 hostname 也是 host 的。直接使用 Docker host 的网络最大的好处就是性能，
不过容器会占用主机的端口。Docker host 的另一个用途是让容器可以直接配置 host 网路。比如某些跨 host 的网络解决方案，
其本身也是以容器方式运行的，这些方案需要对网络进行配置，比如管理 iptables

**bridge网络**

Docker 安装时会创建一个 命名为 docker0 的网桥。如果不指定--network，创建的容器默认都会挂到 docker0 上。

比如我创建一个容器看看

```bash
docker run --name httpd -d httpd
```

然后看一下网络接口：

```bash
[root@ecs-hw01 ~]# brctl show
bridge name     bridge id           STP enabled	    interfaces
docker0         8000.024206a84de0   no              veth5c8e660
```

一个新的网络接口 veth5c8e660 被挂到了 docker0 上，veth5c8e660就是新创建容器的虚拟网卡。

下面看一下容器的网络配置。

```bash
[root@9e0341cf45b8 /]# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
145: eth0@if146: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default 
    link/ether 02:42:ac:11:00:02 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 172.17.0.2/16 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::42:acff:fe11:2/64 scope link 
       valid_lft forever preferred_lft forever
```

容器有一个网卡 eth0@if146。大家可能会问了，为什么不是 veth5c8e660 呢？

实际上 eth0@if146 和 veth5c8e660 是一对 veth pair。veth pair 是一种成对出现的特殊网络设备，
可以把它们想象成由一根虚拟网线连接起来的一对网卡，网卡的一头（eth0@if146）在容器中，另一头（veth5c8e660）挂在网桥 docker0 上，
其效果就是将 eth0@if146 也挂在了 docker0 上。

我们还看到 eth0@if146 已经配置了 IP 172.17.0.2，为什么是这个网段呢？让我们通过 docker network inspect bridge 看一下 bridge
网络的配置信息:

```bash
[root@ecs-hw01 ~]# docker network inspect bridge
[
    {
        "Name": "bridge",
        "Id": "62b50083925d00339f0c9a017f12e6b24b87fb524ee4ffaddeaa11702d298b10",
        "Created": "2019-01-20T16:31:16.167687604+08:00",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": null,
            "Config": [
                {
                    "Subnet": "172.17.0.0/16",
                    "Gateway": "172.17.0.1"
                }
            ]
        },
...
```

原来 bridge 网络配置的 subnet 就是 172.17.0.0/16，并且网关是 172.17.0.1。这个网关在哪儿呢？大概你已经猜出来了，就是 docker0。

当前容器网络拓扑结构如图所示：

![](https://xnstatic-1253397658.file.myqcloud.com/docker-network20.jpg)

## 自定义网络

除了 none, host, bridge 这三个自动创建的网络，用户也可以根据业务需要创建 user-defined 网络。

Docker 提供三种自定义网络驱动：bridge, overlay 和 macvlan。`overlay` 和 `macvlan` 用于创建跨主机的网络，后面有章节单独讨论。

我们可通过 bridge 驱动创建类似前面默认的 bridge 网络，例如：

```bash
docker network create --driver bridge my_net
afc81d275d293b5c978b3a57a71901a030874c4f86d4b61fb6f6bad2183b3c9a
```

然后再看一下主机现在的网络：

```bash
[root@VM_22_2_centos docker]# clear
[root@VM_22_2_centos docker]# ip l
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP mode DEFAULT qlen 1000
    link/ether 52:54:00:cf:74:9f brd ff:ff:ff:ff:ff:ff
3: docker0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT 
    link/ether 02:42:d5:1d:92:2c brd ff:ff:ff:ff:ff:ff
29: veth68c4395@if28: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master docker0 state UP mode DEFAULT 
    link/ether ee:92:96:50:92:87 brd ff:ff:ff:ff:ff:ff link-netnsid 0
32: br-afc81d275d29: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN mode DEFAULT 
    link/ether 02:42:2b:e4:94:71 brd ff:ff:ff:ff:ff:ff
```

多了一个网桥`br-afc81d275d29`，afc81d275d29正是我们创建的网络短id。

执行 `docker network inspect my_net` 查看一下 `my_net` 的配置信息

```
[
    {
        "Name": "my_net",
        "Id": "afc81d275d293b5c978b3a57a71901a030874c4f86d4b61fb6f6bad2183b3c9a",
        "Created": "2018-05-24T18:24:28.734177403+08:00",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": {},
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
        "Containers": {},
        "Options": {},
        "Labels": {}
    }
]

```

这里 172.18.0.0/16 是 Docker 自动分配的 IP 网段。

如果要自己指定IP网段，可使用 --subnet 和 --gateway 参数。

```bash
[root@VM_22_2_centos docker]# docker network create --driver bridge --subnet 172.22.16.0/24 --gateway 172.22.16.1 my_net2
5e9a17690deaef1f3e7c0a12b60a9fb5999a11fb11462dd0b40c6f8ae1e6e970

[root@VM_22_2_centos docker]# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP qlen 1000
    link/ether 52:54:00:cf:74:9f brd ff:ff:ff:ff:ff:ff
    inet 10.122.22.2/24 brd 10.122.22.255 scope global eth0
       valid_lft forever preferred_lft forever
3: docker0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP 
    link/ether 02:42:d5:1d:92:2c brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0
       valid_lft forever preferred_lft forever
29: veth68c4395@if28: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master docker0 state UP 
    link/ether ee:92:96:50:92:87 brd ff:ff:ff:ff:ff:ff link-netnsid 0
32: br-afc81d275d29: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN 
    link/ether 02:42:2b:e4:94:71 brd ff:ff:ff:ff:ff:ff
    inet 172.18.0.1/16 brd 172.18.255.255 scope global br-afc81d275d29
       valid_lft forever preferred_lft forever
33: br-5e9a17690dea: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN 
    link/ether 02:42:cb:d4:46:06 brd ff:ff:ff:ff:ff:ff
    inet 172.22.16.1/24 brd 172.22.16.255 scope global br-5e9a17690dea
       valid_lft forever preferred_lft forever

[root@VM_22_2_centos docker]# ifconfig br-5e9a17690dea
br-5e9a17690dea: flags=4099<UP,BROADCAST,MULTICAST>  mtu 1500
        inet 172.22.16.1  netmask 255.255.255.0  broadcast 172.22.16.255
        ether 02:42:cb:d4:46:06  txqueuelen 0  (Ethernet)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```

容器要使用新的网络，需要在启动时通过 --network 指定：

```bash
[root@VM_22_2_centos docker]# docker run -it --network my_net2 busybox
/ # ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
34: eth0@if35: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue 
    link/ether 02:42:ac:16:10:02 brd ff:ff:ff:ff:ff:ff
    inet 172.22.16.2/24 brd 172.22.16.255 scope global eth0
       valid_lft forever preferred_lft forever
```

还能根据--ip指定静态ip地址：

```bash
[root@VM_22_2_centos docker]# docker run -it --network my_net2 --ip 172.22.16.5 busybox
/ # ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
36: eth0@if37: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue 
    link/ether 02:42:ac:16:10:05 brd ff:ff:ff:ff:ff:ff
    inet 172.22.16.5/24 brd 172.22.16.255 scope global eth0
       valid_lft forever preferred_lft forever
/ # 
```

注：只有使用 --subnet 创建的网络才能指定静态 IP

## 容器之间网络互连

上面两个busybox容器因为都连着my_net2网桥，所以在同一网段，网络可相互ping通。

如果不同时连着同一个网桥，就不能ping通了。比如httpd连的是docker0网桥，而busybox连接的是my_net2网络，那此时怎么办呢？

答案就是为httpd容器添加my_net2网桥。

```bash
docker network connect my_net2 33077e568886
```

然后去httpd容器中看看网络情况：

```bash
docker exec -it 33077e568886 bash
root@33077e568886:/usr/local/apache2# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
40: eth0@if41: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default 
    link/ether 02:42:ac:11:00:03 brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.3/16 brd 172.17.255.255 scope global eth0
       valid_lft forever preferred_lft forever
48: eth1@if49: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default 
    link/ether 02:42:ac:16:10:02 brd ff:ff:ff:ff:ff:ff
    inet 172.22.16.2/24 brd 172.22.16.255 scope global eth1
       valid_lft forever preferred_lft forever
root@33077e568886:/usr/local/apache2# 
```

容器中增加了一个网卡 eth1，分配了 my_net2 的 IP 172.22.16.2。现在 busybox 应该能够访问 httpd 了，验证一下：

```bash
/ # ping -c 3 172.22.16.2
PING 172.22.16.2 (172.22.16.2): 56 data bytes
64 bytes from 172.22.16.2: seq=0 ttl=64 time=0.134 ms
64 bytes from 172.22.16.2: seq=1 ttl=64 time=0.093 ms
64 bytes from 172.22.16.2: seq=2 ttl=64 time=0.111 ms

--- 172.22.16.2 ping statistics ---
3 packets transmitted, 3 packets received, 0% packet loss

/ # wget 172.22.16.2
Connecting to 172.22.16.2 (172.22.16.2:80)
index.html           100% |*********************************************|    45   0:00:00 ETA
/ # cat index.html 
<html><body><h1>It works!</h1></body></html>
/ # 
```

busybox 能够 ping 到 httpd，并且可以访问 httpd 的 web 服务。

现在的网络拓扑结构如下图：

![](https://xnstatic-1253397658.file.myqcloud.com/docker32.png)

## 三种通信方式

上一节讲了如何让容器处在同一个网络之下，也就是使用Docker Networking的方式，这也是推荐方式。
本节讲一下同一网络下的容器通信的几种方式。容器之间可通过 IP，Docker DNS Server 或 joined 容器三种方式通信。

**IP 通信**

只需要连在同一个网桥上面在同一个网段就能够相互通过ip地址通信，这个最简单。

**Docker DNS Server**

通过 IP 访问容器虽然满足了通信的需求，但还是不够灵活。因为我们在部署应用之前可能无法确定 IP，
部署之后再指定要访问的 IP 会比较麻烦。对于这个问题，可以通过 docker 自带的 DNS 服务解决。

只需要在启动容器的时候通过 --name 指定容器名即可。

使用 docker DNS 有个限制：只能在用户自定义网络中使用。也就是说，默认的 bridge 网络是无法使用 DNS 的。

**joined 容器**

joined 容器非常特别，它可以使两个或多个容器共享一个网络栈，共享网卡和配置信息，joined 容器之间可以通过 127.0.0.1 直接通信。

看一个例子，先创建一个 httpd 容器，名字为 web1：

```bash
docker run -d -it --name=web1 httpd
```

然后创建 busybox 容器并通过 `--network=container:web1` 指定 jointed 容器为 web1：

```bash
[root@VM_22_2_centos ~]# docker run -it --network=container:web1 busybox
/ # ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
52: eth0@if53: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue 
    link/ether 02:42:ac:11:00:04 brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.4/16 brd 172.17.255.255 scope global eth0
       valid_lft forever preferred_lft forever
/ # 
```

请注意 busybox 容器中的网络配置信息。下面我们查看一下 web1 的网络：

```bash
root@f0c9d4aa4a88:/usr/local/apache2# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
52: eth0@if53: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default 
    link/ether 02:42:ac:11:00:04 brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.4/16 brd 172.17.255.255 scope global eth0
       valid_lft forever preferred_lft forever
```

busybox 和 web1 的网卡 mac 地址与 IP 完全一样，它们共享了相同的网络栈。busybox 可以直接用 127.0.0.1 访问 web1 的 http 服务

```bash
/ # wget 127.0.0.1
Connecting to 127.0.0.1 (127.0.0.1:80)
index.html           100% |*****************************************|    45   0:00:00 ETA
/ # cat index.html 
<html><body><h1>It works!</h1></body></html>
```

joined 容器非常适合以下场景：

1. 不同容器中的程序希望通过 loopback 高效快速地通信，比如 web server 与 app server。
1. 希望监控其他容器的网络流量，比如运行在独立容器中的网络监控程序。

## 容器访问外部世界

前面我们已经解决了容器间通信的问题，接下来讨论容器如何与外部世界通信。这里涉及两个方向：

1. 容器访问外部世界
1. 外部世界访问容器

通过 NAT，docker 实现了容器对外网的访问。

如果网桥 docker0 收到来自 172.17.0.0/16 网段的外出包，把它交给 MASQUERADE 处理。
而 MASQUERADE 的处理方式是将包的源地址替换成 host 的地址发送出去，即做了一次网络地址转换（NAT）。

## 外部世界访问容器

方法就是端口映射。

docker 可将容器对外提供服务的端口映射到 host 的某个端口，外网通过该端口访问容器。容器启动时通过-p参数映射端口：

```bash
[root@VM_22_2_centos docker]# docker run -d -p 8080:80 --name=web2 httpd
e11b5daec62e974abf7aa94665889956bab5b1b6eb167acbdc880bdf7c15a5e6

[root@VM_22_2_centos docker]# wget localhost:8080
--2018-05-24 19:28:45--  http://localhost:8080/
Resolving localhost (localhost)... 127.0.0.1, ::1
Connecting to localhost (localhost)|127.0.0.1|:8080... connected.
HTTP request sent, awaiting response... 200 OK
Length: 45 [text/html]
Saving to: ‘index.html’

100%[===============================================================>] 45          --.-K/s   in 0s      

2018-05-24 19:28:45 (7.69 MB/s) - ‘index.html’ saved [45/45]

[root@VM_22_2_centos docker]# cat index.html 
<html><body><h1>It works!</h1></body></html>
[root@VM_22_2_centos docker]# 

```

可以看到，上面讲主机的8080端口映射到容器的80端口，然后通过`wget localhost:8080`就能下载到容器中web首页。

每一个映射的端口，host 都会启动一个 docker-proxy 进程来处理访问容器的流量

```bash
[root@VM_22_2_centos docker]# ps -ef |grep docker-proxy
root     12302  6752  0 19:28 ?        00:00:00 /usr/bin/docker-proxy -proto tcp -host-ip 0.0.0.0 -host-port 8080 -container-ip 172.17.0.4 -container-port 80
```

