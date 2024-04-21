---
title: Docker每天学一点01 - 入门
date: 2019-03-01 16:11:23
toc: true
categories: [ k8s ]
tags: [ docker ]
---

容器是一种轻量级、可移植、自包含的软件打包技术，使应用程序可以在几乎任何地方以相同的方式运行。
开发人员在自己笔记本上创建并测试好的容器，无需任何修改就能够在生产系统的虚拟机、物理服务器或公有云主机上运行。

可以将容器想象成运输行业中的集装箱，Docker 将集装箱思想运用到软件打包上，为代码提供了一个基于容器的标准化运输系统。
Docker 可以将任何应用及其依赖打包成一个轻量级、可移植、自包含的容器。容器可以运行在几乎所有的操作系统上。

其实，"集装箱" 和 "容器" 对应的英文单词都是 "Container"。"容器" 是国内约定俗成的叫法，
可能是因为容器比集装箱更抽象，更适合软件领域的原故吧。Docker 的 Logo 不就是一条鲸鱼上面一堆集装箱吗？

第一篇我先搭建实验环境，尽快让一个容器运行起来，我使用的操作系统是CentOS7.2，安装的是免费的社区版CE。
<!-- more -->

## 安装Docker

```
# step 1: 安装必要的一些系统工具
sudo yum install -y yum-utils device-mapper-persistent-data lvm2
# Step 2: 添加软件源信息
sudo yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
# Step 3: 更新并安装 Docker-CE
sudo yum makecache fast
sudo yum -y install docker-ce
```

## 配置文件

默认没有这些配置文件，手动创建即可

* /etc/default/docker ==> service启动配置文件
* /etc/docker/daemon.json ==> 1.12版本后万能配置文件

编辑`/etc/docker/daemon.json`，如果没有就创建一个：

```
sudo mkdir /etc/docker
vim /etc/docker/daemon.json
```

写入如下内容：

```
{
  "storage-driver": "devicemapper"
}
```

## 启动docker

```
sudo systemctl start docker
```

查看启动状态：

```
sudo systemctl status docker
```

## 镜像下载加速

由于 Docker Hub 的服务器在国外，下载镜像会比较慢。幸好 DaoCloud 为我们提供了免费的国内镜像服务。

下面介绍如果使用镜像服务。

在 daocloud.io 免费注册一个用户，可直接用GitHub用户注册。进入加速器页面: <https://www.daocloud.io/mirror>

对于linux平台，下面有一行脚本：

```
curl -sSL https://get.daocloud.io/daotools/set_mirror.sh | sh -s http://f1361db2.m.daocloud.io
```

我测试的时候，这个脚本会在/etc/docker/daemon.json中写入：

```
{"registry-mirrors": ["http://f1361db2.m.daocloud.io"],}
```

请把里面的逗号去掉，然后重启docker，体验飞一般的感觉：

```
sudo systemctl restart docker.service
```

## 测试docker

通过运行`hello world`镜像来测试容器是否已经被成功安装：

```
sudo docker run hello-world
```

这条命令会现在一个test镜像并在容器里运行它，当容器运行后会打印一条信息并退出来。

或者运行一个httpd镜像：

```
docker run -d -p 8020:80 httpd
```

从 Docker Hub 下载 httpd 镜像。镜像中已经安装好了 Apache HTTP Server，并将容器的 80 端口映射到 host 的 8020 端口

然后访问：`http://[hostIP]`就可以看到效果了：

![](https://xnstatic-1253397658.file.myqcloud.com/docker01.png)
