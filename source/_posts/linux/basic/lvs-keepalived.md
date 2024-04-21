---
title: CentOS6.5上LVS和KeepAlived搭建高可用负载均衡集群
date: 2015-09-16 16:40:04 +0800
toc: true
categories: [ linux ]
tags: [ linux ]
---

我们不仅要知其然，而且要知其所以然，所以先给大家准备一些理论知识课，这样对以后的应用将会事半功倍。

**1、什么是LVS？**

请阅读作者章文嵩博士自己的研究报告，共计4部分，看完后对集群和LVS就有了初步的了解，不懂时可以翻翻。

+ LVS项目介绍：<http://www.linuxvirtualserver.org/zh/lvs1.html>
+ LVS集群的体系结构：<http://www.linuxvirtualserver.org/zh/lvs2.html>
+ LVS集群中的IP负载均衡技术：<http://www.linuxvirtualserver.org/zh/lvs3.html>
+ LVS集群的负载调度：<http://www.linuxvirtualserver.org/zh/lvs4.html>

<!-- more -->

**2、什么是KeepAlived?**

Keepalived原理与实战精讲： <http://zhumeng8337797.blog.163.com/blog/static/100768914201191762253640/>

**3、什么是CentOS?**

百度百科给出的答案：[什么是CentOS][]

**4、小结**

相信读了以上的理论知识后，已经对集群的实现原理有了大概的了解，那接下来我们就开始动手吧。

系统架构图：

![](https://xnstatic-1253397658.file.myqcloud.com/0002.png)

## 服务器的安装

我们会用到4个服务器，横向分2层：

第1层是LVS服务器（1个主，1个从；从可以多个）用来转发请求，需要安装ipvsadm和keepAlived；
第2层是提供具体服务的服务器（表中用了2个；当然也可以是多个，现实的应用会上百台。），
安装的是具体的服务，这里我们安装的是TOMCAT。

主机环境如下：

```
192.168.203.107　 LVS_VIP（VIP：Virtual IP）
192.168.203.103　 LVS_Master　　
192.168.203.104　 LVS_Backup
192.168.203.93　  WEB1_RealServer
192.168.203.94　  WEB2_RealServer
```

克隆：我们先安装配置好一层的一个服务器，其他服务器使用克隆方式。

### 1\. 安装虚拟机VirtualBox4.2.12

### 2\. 安装CentOS 6.5

这一步先安装一台虚拟机，然后其他的通过克隆方式安装，不过注意的是，
克隆之后需要修改相应的IP地址已经eth0等配置。具体方法是：

#### 1) 修改/etc/udev/rules.d/70-persistent-net.rules

拷贝eth1的硬件地址到eth0，删除eth1信息

#### 2) 配置/etc/sysconfig/network-scripts/ifcfg-eth0

```
DEVICE="eth0"
BOOTPROTO="static"
HWADDR="00:0C:29:91:42:2C"
MTU="1500"
NM_CONTROLLED="yes"
ONBOOT="yes"
IPADDR=192.168.203.94
NETMASK=255.255.255.0
GATEWAY=192.168.203.254
```

#### 3) reboot

这样可以保证所有的机器网卡都是eth0接口

### 3\. LVS层安装LVS和KeepAlived

首先是LVS_MASTER机器的安装配置，打开LVS_Master服务器；

先安装lvs_master的服务，lvs_backup使用克隆虚拟机的方式，然后在配置文件修改三个参数即可，下面会讲到。

#### 1) 安装IPVSADM

知识点：IPVSADM理解为IPVS管理工具；LVS（Linux Virtual Server）的核心为IPVS（IP Virtual Server），
从Linux内核版本2.6起，IPVS模块已经编译进了Linux内核。

使用yum命令进行安装，系统会选择最适合内核版本的ipvsadm

```
yum -y install ipvsadm
```

#### 2) 防火墙

为了测试方便，我们直接关闭防火墙，在实际使用中开启需要的端口即可。

具体配置可参考：<http://www.cnblogs.com/rockee/archive/2012/05/17/2506671.html>

```
service iptables stop
```

#### 3) KeepAlived 的安装

知识点：KeepAlived是一个路由软件，它主要的目的是让我们通过简单的配置，实现高可用负载均衡，
当然负载均衡依赖于Linux虚拟服务器（IPVS）的内核模块，其高可用性使用VRRP协议来实现，
KeepAlived不仅会检测负载均衡服务器池中每台机器的健康状况并通知IPVS将非健康的机器从池中移除掉；
同时它还能对负载均衡调度器本身实现健康状态检查，当主负载均衡调度器出现问题时，备用负载均衡调度器顶替主进行工作。
逐条执行如下命令，执行的原因暂不解释，实际就是需要这些组件，安装即可。

```bash
cd /usr/src
yum -y install openssl-devel
wget http://www.keepalived.org/software/keepalived-1.2.7.tar.gz
wget http://mirror.centos.org/centos/6/os/x86_64/Packages/popt-static-1.13-7.el6.x86_64.rpm
yum -y install popt-static-1.13-7.el6.x86_64.rpm
yum -y install kernel-devel make gcc openssl-devel libnl* popt*
ln -s /usr/src/kernels/2.6.32-220.13.1.el6.x86_64/ /usr/src/linux
tar zxvf keepalived-1.2.7.tar.gz
cd keepalived-1.2.7
./configure --with-kernel-dir=/usr/src/kernels/2.6.32-358.2.1.el6.x86_64/
make && make install
cp /usr/local/etc/rc.d/init.d/keepalived /etc/rc.d/init.d/
cp /usr/local/etc/sysconfig/keepalived /etc/sysconfig/
mkdir /etc/keepalived
cp /usr/local/etc/keepalived/keepalived.conf /etc/keepalived/
cp /usr/local/sbin/keepalived /usr/sbin/
```

注：上面的kernel路径自己去用tab键弄出来
OK，KeepAlived安装完毕，然后进行配置。

#### 4) KeepAlivde的配置

**第一步：打开IP Forward 功能**

（LVS现有三种负载均衡规则都需要打开此功能，如果不打开此功能，下面的配置配得再好都无济于事）

```bash
vim /etc/sysctl.conf
```

打开后修改里面"net.ipv4.ip_forward = 1"，修改好后保存退出，执行如下命令使设置立即生效

```bash
sysctl -p
```

**第二步：KeepAlivde的配置**

配置文件在这个位置： /etc/keepalived/keepalived.conf

启动KeepAlived时，它默认会去/etc/keepalived下面找它的配置文件，
所以上面命令中我们已经将这个配置文件复制过来了。现在进行修改：

```
! Configuration File for keepalived

global_defs {
   notification_email {
     test@sina.com
   }
   notification_email_from admin@test.com
   smtp_server 127.0.0.1
   smtp_connect_timeout 30
   router_id LVS_MASTER
}

vrrp_instance VI_1 {
    state MASTER
    interface eth0
    virtual_router_id 60
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        192.168.203.107
    }
}

virtual_server 192.168.203.107 8080 {
    delay_loop 6
    lb_algo rr
    lb_kind DR
    nat_mask 255.255.255.0
    persistence_timeout 50
    protocol TCP

    real_server 192.168.203.93 8080 {
        weight 1
        TCP_CHECK {
            connect_timeout 3
            nb_get_retry 3
            delay_before_retry 3
        }
    }

    real_server 192.168.203.94 8080 {
        weight 1
        TCP_CHECK {
            connect_timeout 3
            nb_get_retry 3
            delay_before_retry 3

    }
}
```

以上就完成了keepAlived的配置，下面进行启动

```bash
chkconfig keepalived on
service keepalived start
```

查看进程

```bash
ps aux | grep keepalived
```

Keepalived正常运行时，共启动3个进程，其中一个进程是父进程，负责监控其子进程；
一个是vrrp子进程；另外一个是checkers子进程。

查看下虚拟IP是否已经加上（重要）

```
[root@centos03 keepalived-1.2.7]# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 16436 qdisc noqueue state UNKNOWN
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP qlen 1000
    link/ether 08:00:27:81:29:7d brd ff:ff:ff:ff:ff:ff
    inet 192.168.203.103/24 brd 192.168.203.255 scope global eth0
    inet 192.168.203.107/32 scope global eth0
    inet6 fe80::a00:27ff:fe81:297d/64 scope link
       valid_lft forever preferred_lft forever
```

说明虚拟IP已经自动配置上了。

还有3个命令在先列示下，并不用执行

* 显示集群中服务器ip信息：`ipvsadm -ln`
* 查看日志：`tail -f /var/log/messages`
* 查看请求转发情况：`ipvsadm -lcn | grep 虚拟IP`

至此，LVS_MASTER服务器已经配置好并启动了，接下来我们配置web服务器。

#### 5) WEB服务器WEB1_RealServer的配置

1\. 克隆虚拟机WEB1_RealServer(192.168.203.93)；
2\. 配置虚拟IP启动脚本

```bash
vim /etc/init.d/realserver.sh
```

在文件中输入如下脚本：

```bash
#!/bin/bash
SNS_VIP=192.168.203.107
. /etc/rc.d/init.d/functions
case "$1" in
start)
 ifconfig lo:0 $SNS_VIP netmask 255.255.255.255 broadcast $SNS_VIP
 /sbin/route add -host $SNS_VIP dev lo:0
 echo "1" >/proc/sys/net/ipv4/conf/lo/arp_ignore
 echo "2" >/proc/sys/net/ipv4/conf/lo/arp_announce
 echo "1" >/proc/sys/net/ipv4/conf/all/arp_ignore
 echo "2" >/proc/sys/net/ipv4/conf/all/arp_announce
 sysctl -p >/dev/null 2>&1
 echo "RealServer Start OK"
 ;;
stop)
 ifconfig lo:0 down
 route del $SNS_VIP >/dev/null 2>&1
 echo "0" >/proc/sys/net/ipv4/conf/lo/arp_ignore
 echo "0" >/proc/sys/net/ipv4/conf/lo/arp_announce
 echo "0" >/proc/sys/net/ipv4/conf/all/arp_ignore
 echo "0" >/proc/sys/net/ipv4/conf/all/arp_announce
 echo "RealServer Stoped"
 ;;
 *)
 echo "Usage: $0 {start|stop}"
 exit 1
esac
exit 0
```

3\. 安装配置TOMCAT

我测试用的是TOMCAT6

```bash
yum -y install tomcat6 tomcat6-webapps tomcat6-admin-webapps
service tomcat6 start
```

关闭防火墙

```bash
service iptables stop
```

打开浏览器：http://192.168.203.93:8080

会看到TOMCAT的熟悉页面了。

为了测试负载均衡，我们将这个页面改下，以更好的标识这个网页是本服务器的

Tomcat6安装目录位于/usr/share/tomcat6，所以我们要编辑tomcat下的webapps/ROOT/index.html这个文件。

```bash
cd /usr/share/tomcat6/webapps/ROOT/
cat /dev/null > index.html
vim index.html
```

将如下文本写入index.html，然后打开浏览器：<http://192.168.203.93:8080>，已经改变：

```
web1 192.168.203.93
```

+ 启动虚拟IP的脚本

```bash
sh /etc/init.d/realserver.sh start
ifconfig
```

运行后会看到网络有一个虚拟IP：

```
[root@centos01 ROOT]# ifconfig
eth0      Link encap:Ethernet  HWaddr 08:00:27:59:AB:1D
          inet addr:192.168.203.93  Bcast:192.168.203.255  Mask:255.255.255.0
          inet6 addr: fe80::a00:27ff:fe59:ab1d/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:81006 errors:0 dropped:0 overruns:0 frame:0
          TX packets:12305 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000
          RX bytes:83023415 (79.1 MiB)  TX bytes:1645604 (1.5 MiB)

lo        Link encap:Local Loopback
          inet addr:127.0.0.1  Mask:255.0.0.0
          inet6 addr: ::1/128 Scope:Host
          UP LOOPBACK RUNNING  MTU:16436  Metric:1
          RX packets:46 errors:0 dropped:0 overruns:0 frame:0
          TX packets:46 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:3559 (3.4 KiB)  TX bytes:3559 (3.4 KiB)

lo:0      Link encap:Local Loopback
          inet addr:192.168.203.107  Mask:255.255.255.255
          UP LOOPBACK RUNNING  MTU:16436  Metric:1
```

4\. 去LVS_MASTER服务器的终端查看下ipvsadm，查看已经连接上了WEB1服务器，运行命令

```bash
ipvsadm -ln
```

结果如下：

```
[root@centos03 keepalived-1.2.7]# ipvsadm -ln
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  192.168.203.107:8080 rr persistent 50
  -> 192.168.203.93:8080          Route   1      0          0
```

已经可以看到有服务器加入进来了。
此时我们访问网页<http://192.168.203.107:8080>，出现界面显示web1 192.168.203.93；
OK，至此已经能实现负载均衡了，接下来我们通过克隆实现多个主机的试验。

5\. 服务器克隆

1) 从LVS_MASTER克隆一个LVS_BACKUP服务器，然后修改其中的参数，
   MASTER与BACKUP配置仅三处不同：global_defs中的router_id、vrrp_instance中的state、priority
   （注意keepAlived的配置文件中有一个网卡设备，虚拟机的网卡设备可能是不一样的，
   有的是eth0，有的是eth1，所以也是要改动的，否则从服务器的服务器很有可能服务不正常）

配置好的如下文：

```
! Configuration File for keepalived

global_defs {
   notification_email {
     test@sina.com
   }
   notification_email_from admin@test.com
   smtp_server 127.0.0.1
   smtp_connect_timeout 30
   router_id LVS_BACKUP
}

vrrp_instance VI_1 {
    state BACKUP
    interface eth0
    virtual_router_id 60
    priority 90
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        192.168.203.107
    }
}

virtual_server 192.168.203.107 8080 {
    delay_loop 6
    lb_algo rr
    lb_kind DR
    nat_mask 255.255.255.0
    persistence_timeout 50
    protocol TCP

    real_server 192.168.203.93 8080 {
        weight 1
        TCP_CHECK {
            connect_timeout 3
            nb_get_retry 3
            delay_before_retry 3
        }
    }

    real_server 192.168.203.94 8080 {
        weight 1
        TCP_CHECK {
            connect_timeout 3
            nb_get_retry 3
            delay_before_retry 3
        }
    }
}
```

2) 从WEB1_RealServer克隆一个WEB2_RealServer，将tomcat的index.html文件改为web2 192.168.203.94。
   （这里的IP是我测试的，您的可以自定义）启动realserver.sh脚本。

3) OK，至此我们已经虚拟出2个LVS服务器，一对主从；2个WEB服务器，web1和web2。
   接下来我们进行测试，看能否满足我们的初始需求。

## 负载和可用性测试

开启每个服务器的相关服务，关闭防火墙，我们开始进行测试。

测试LVS层

1）首先执行ip a命令，主服务器会存在一个虚拟IP，从服务器不会存在这个虚拟IP。现在浏览网页显示正常。虚拟IP如图所示：

```
显示集群中服务器ip信息：ipvsadm -ln
查看日志：tail -f /var/log/messages
查看请求转发情况：ipvsadm -lcn | grep 虚拟IP
```

2）现在停掉LVS_MASTER的keepAlived服务，看LVS_BACKUP是否可以自动加上虚拟IP地址，并且开始转发请求。

（注意keepAlived的配置文件中有一个网卡设备，虚拟机的网卡设备可能是不一样的，有的是eth0，
有的是eth1，所以也是要改动的，否则从服务器的服务器很有可能服务不正常）
之后你通过命令：ip a去分别查看LVS_MASTER和LVS_BACKUP机器，结果正如预料一样，BACKUP立即接管了MASTER的工作。
切换很快，访问网页：<http://192.168.203.107:8080> 也能正常显示。

3）恢复主服务器的keepAlived服务后，主服务器立刻接替了从服务器的工作，就不做截图了。和第1）个正常效果是一样的。

4）测试WEB服务器，看能否正常提供服务。先断掉WEB1，看下效果。

ipvsadm中的服务器列表，已经去掉了WEB1服务器，访问网页也只能访问到WEB2服务器了。

5）开启WEB1，关掉WEB2。测试正常。

## 总结

经过不断的测试，终于完成了这篇稿子，望大家能够指正。还有一点就是很多时候都是配置文件中的一些小毛病造成的，比如：

1. keepAlived中的通知邮箱好像必须要写，否则不正确；
1. keepAlived中的网卡设备要注意，按照服务器的实际情况填写；
1. 使用时，必要的端口要打开，或者关掉防火墙。否则有事不提供服务；
1. 一些命令行的执行，少一些参数执行就可能会有一些问题。
1. LINUX系统的目录结构也头疼，要不断的熟悉，否则也让你故意弄混了。

—————————————————全文完——————————————–

[什么是CentOS]: http://baike.baidu.com/link?url=X3SzN3bJWjW_PkWC6GB2MTs5KhVmxBAxnCRjs9W7-IARDiHloZ_oRWj17BEz0kY3
