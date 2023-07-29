---
title: CentOS7搭建NFS网络文件系统
date: 2018-02-19 22:46:01 +0800
toc: true
categories: [ linux ]
tags: [ linux ]
---

如果需要共享文件的主机都是Linux系统，强烈推荐大家在客户端部署NFS服务来共享文件。
NFS（网络文件系统）服务可以将远程Linux系统上的文件共享资源挂载到本地主机的目录上，
从而使得本地主机（Linux客户端）基于TCP/IP协议，像使用本地主机上的资源那样读写远程Linux系统上的共享文件。

为了检验NFS服务配置的效果，我们需要使用两台Linux主机（一台充当NFS服务器，一台充当NFS客户端）。
下面的演示中，master主机为服务端，host1主机为客户端。
<!-- more -->

## 服务端配置

安装NFS服务

```bash
yum install -y nfs-utils
```

另外，不要忘记配置防火墙策略，以免默认的防火墙策略禁止正常的NFS共享服务。

```bash
firewall-cmd --zone=public --permanent --add-service={rpc-bind,mountd,nfs}
firewall-cmd --reload
```

在NFS服务器上建立用于NFS文件共享的目录，并设置足够的权限确保其他人也有写入权限。

```bash
[root@master ~]#  mkdir /nfsfile
[root@master ~]# chmod -Rf 777 /nfsfile
[root@master ~]# echo "welcome to xncoding.com" > /nfsfile/readme
```

NFS服务程序的配置文件为`/etc/exports`，默认情况下里面没有任何内容。
我们可以按照`共享目录的路径 允许访问的NFS客户端（共享权限参数）`的格式，定义要共享的目录与相应的权限。

例如，如果想要把/nfsfile目录共享给192.168.1.0/24网段内的所有主机，让这些主机都拥有读写权限，
在将数据写入到NFS服务器的硬盘中后才会结束操作，最大限度保证数据不丢失，
以及把来访客户端root管理员映射为本地的匿名用户等，则可以按照下面命令中的格式，
将下表中参数写到NFS服务程序的配置文件中。

 参数             | 说明                                   
----------------|--------------------------------------
 ro             | 只读                                   
 rw             | 读写                                   
 root_squash    | 当NFS客户端以root管理员访问时，映射为NFS服务器的匿名用户    
 no_root_squash | 当NFS客户端以root管理员访问时，映射为NFS服务器的root管理员 
 all_squash     | 无论NFS客户端使用什么账户访问，均映射为NFS服务器的匿名用户     
 no_all_squash  | 当NFS客户端以普通用户访问时，映射为NFS服务器的普通用户       
 sync           | 同时将数据写入到内存与硬盘中，保证不丢失数据               
 async          | 优先将数据保存到内存，然后再写入硬盘；这样效率更高，但可能会丢失数据   

请注意，NFS客户端地址与权限之间没有空格。

```
[root@master ~]# vim /etc/exports
/nfsfile 192.168.1.*(rw,sync,root_squash)
```

启动和启用NFS服务程序。由于在使用NFS服务进行文件共享之前，
需要使用RPC（Remote Procedure Call，远程过程调用）服务将NFS服务器的IP地址和端口号等信息发送给客户端。
因此，在启动NFS服务之前，还需要顺带重启并启用rpcbind服务程序，并将这两个服务一并加入开机启动项中。

```bash
[root@master ~]# systemctl restart rpcbind
[root@master ~]# systemctl enable rpcbind
[root@master ~]# systemctl start nfs-server
[root@master ~]# systemctl enable nfs-server
Created symlink from /etc/systemd/system/multi-user.target.wants/nfs-server.service to /usr/lib/systemd/system/nfs-server.service.
```

## 客户端配置

客户端也需要安装nfs-utils这个软件包：

```bash
yum install -y nfs-utils
```

NFS客户端的配置步骤也十分简单。先使用`showmount`命令（以及必要的参数）查询NFS服务器的远程共享信息，
其输出格式为`共享的目录名称 允许使用客户端地址`。

`showmount`命令中可用的参数以及作用

 参数 | 说明                
----|-------------------
 -e | 显示NFS服务器的共享列表     
 -a | 显示本机挂载的NFS文件资源的情况 
 -v | 显示版本号             

```bash
[root@host1 ~]# showmount -e 192.168.1.20
Export list for 192.168.1.20:
/nfsfile 192.168.1.*
```

然后在NFS客户端创建一个挂载目录。使用`mount`命令并结合`-t`参数，
指定要挂载的文件系统的类型，并在命令后面写上服务器的IP地址、
服务器上的共享目录以及要挂载到本地系统（即客户端）的目录。

```bash
[root@host1 ~]# mkdir /nfsfile
[root@host1 ~]# mount -t nfs 192.168.1.20:/nfsfile /nfsfile
[root@host1 ~]# ls -l /nfsfile/
总用量 4
-rw-r--r-- 1 root root 24 8月  26 22:53 readme
[root@host1 ~]# cat /nfsfile/readme 
welcome to xncoding.com
```

挂载成功后就应该能够顺利地看到在执行前面的操作时写入的文件内容了。
如果希望NFS文件共享服务能一直有效，则需要将其写入到fstab文件中，编辑`/etc/fstab`如下：

```
#
# /etc/fstab
# Created by anaconda on Sun Jan 27 21:25:27 2019
#
# Accessible filesystems, by reference, are maintained under '/dev/disk'
# See man pages fstab(5), findfs(8), mount(8) and/or blkid(8) for more info
#
/dev/mapper/centos-root /  xfs     defaults    0 0
UUID=3acaf9d0-4cd0-40d0-af57-c84cd23fec0c /boot  xfs     defaults  0 0
/dev/mapper/centos-swap swap  swap    defaults    0 0
//192.168.1.20/rootdir /database cifs credentials=/root/auth.smb 0 0
192.168.1.20:/nfsfile /nfsfile nfs defaults 0 0
```

然后执行`mount -a`

