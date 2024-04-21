---
title: Vagrant创建虚拟化开发环境
date: 2021-01-22 09:12:33 +0800
toc: true
categories: [ linux ]
tags: [ linux ]
---

作为一名开发者，是否经常抱怨环境问题，某个bug只出现在你的环境下面，找了很久才找到原来是一个很小的环境差异导致。 
Vagrant可以非常容易的配置一个统一的可复制、可移植的分布式开发环境，
在VirtualBox、VMware、AWS或[其他provider](https://www.vagrantup.com/docs/providers/)平台上，
借助[provisioning工具](https://www.vagrantup.com/docs/provisioning/)，
比如shell脚本、Ansible、Chef等自动在各个机器上面安装和配置好软件。

只需要一个Vagrantfile，别人就能基于它创建统一的环境，不管你的工作机器是Linux、Mac OS 还是Windows系统， 
最后创建的虚拟机环境都是一样的。官网：<https://www.vagrantup.com/>
<!-- more -->

## 安装

直接去[下载页面](https://www.vagrantup.com/downloads.html)下载对应版本，
或者通过yum、apt安装最新的随你便。总是安装过程太简单不解释了，我的是Ubuntu16.04 LTS 64-bit版本，选择使用apt安装的。

```bash
sudo apt-get install virtualbox
sudo apt-get install vagrant
```

## 初始化

这里我们使用Virtualbox这个默认的provider来演示，先安装Virtualbox和Vagrant后。

```bash
$ vagrant init hashicorp/precise64
$ vagrant up
```

这条命令会自动去下载一个box并将它加进来，然后基于这个box创建并启动虚拟机。 这样你就在virtualbox里面安装好了一个Ubuntu 12.04 LTS 64-bit的系统。
通过`vagrant ssh`可登陆进去看看。

创建任何vagrant工程的第一步就是创建一个[Vagrantfile](https://www.vagrantup.com/docs/vagrantfile/)。
这个文件有两个作用：

* 决定你项目的根目录，很多配置项路径都是相对于这个根目录的
* 描述机器操作系统类型和需要的资源，以及需要安装在机器上面的所有软件

通过在某个目录下使用`vagrant init`可初始化一个项目

```bash
$ mkdir vagrant_getting_started
$ cd vagrant_getting_started
$ vagrant init
```

一般都会将这个Vagrantfile放入版本控制系统，这样其他人都能拿到这个文件，基于它来创建相同的环境。

## Boxes

并不需要每次都从来开始构建一个系统，Vagrant可使用一个基础镜像来快速克隆一个虚拟机，这个基础镜像就叫"box"。
在生成初始化Vagrantfile后第一步就是指定Box。

如果你按照之前步骤运行过，那么你已经安装了一个box了，下面的步骤就不需要了。

```bash
$ vagrant box add hashicorp/precise64
```

这条命令会从 [HashiCorp's Atlas box catalog](https://atlas.hashicorp.com/boxes/search) 下载名字叫"hashicorp/precise64"
的box并加进来。你还能从本地文件、自定义URL来加入一个box。

当我们把box加入vagrant之后，接下来需要在Vagrantfile中指定要使用它，你还可以指定版本，下载URL等等：

```bash
Vagrant.configure("2") do |config|
  config.vm.box = "hashicorp/precise64"
  config.vm.box_version = "1.1.0"
  config.vm.box_url = "http://files.vagrantup.com/precise64.box"
end
```

## 启动并访问

通过如下命令启动vagrant环境：

```bash
$ vagrant up
```

ssh登录：

```bash
$ vagrant ssh
```

进入之后你别去执行 `rm -rf /` 这个命令就行，因为vagrant虚拟机和你的宿主机共享了一个文件夹/vagrant。

关闭虚拟机环境

```bash
$ vagrant halt
```

销毁vagrant：

```bash
$ vagrant destroy
```

彻底移除box：

```bash
$ vagrant box remove
```

## 同步文件夹

Vagrant可以自动帮你同步宿主机和虚拟机的文件夹。 宿主机的文件夹是你的项目根目录，也就是Vagrantfile文件所在目录；
虚拟机的同步目录默认是/vagrant目录。这样你就能在自己宿主机上使用喜欢的编辑器或开发工具进行开发了，不需要考虑同步问题。

## Provision

先解释下名词。在Vagrant里面有两个重要概念，一个是Provision，一个是Provider

* Provision：远程配置和安装工具，通过它来通过ssh来完成远程虚拟机的各个软件安装和配置，比如shell、Ansible、Chef
* Provider：用来创建虚拟机并为其提供环境和基础设施，比如VirtualBox、VMware、AWS或[其他provider](https://www.vagrantup.com/docs/providers/)平台

通过之前的步骤，你已经有了一个远程环境。现在我要在这个机器上跑web服务，那么需要安装和启动apache软件。这里我选用shell脚本来作为我的provision，
在项目根目录创建一个bootstrap.sh：

```bash
#!/usr/bin/env bash

apt-get update
apt-get install -y apache2
if ! [ -L /var/www ]; then
  rm -rf /var/www
  ln -fs /vagrant /var/www
fi
```

然后，在项目根目录新加一个html文件夹，里面增加index.html，写点内容进去。然后配置Vagrantfile来使用它：

```bash
Vagrant.configure("2") do |config|
  config.vm.box = "hashicorp/precise64"
  config.vm.provision :shell, path: "bootstrap.sh"
end
```

再次运行 `vagrant up` 来初始化和启动你的虚拟机，要是虚拟机之前已经启动过了，只需要执行

```bash
vagrant reload --provision
```

便可很快的重新启动机器，注：Vagrant只在up的第一次执行provisioners。所以要加强制执行参数--provision

ssh登录后看看apache是不是已经正常工作了：

```bash
$ vagrant ssh
...
vagrant@precise64:~$ wget -qO- 127.0.0.1
```

## 网络

上面的服务只能在本机访问，外面访问不了。这里我们再设置下端口转发即可让外面的机器也能访问这个web服务。 编辑Vagrantfile：

```bash
Vagrant.configure("2") do |config|
  config.vm.box = "hashicorp/precise64"
  config.vm.provision :shell, path: "bootstrap.sh"
  config.vm.network :forwarded_port, guest: 80, host: 4567
end
```

运行`vagrant reload` 或者 `vagrant up` (取决于你虚拟机是否已经启动了)，然后在你的宿主机上面访问：http://127.0.0.1:4567

Vagrant还能配置更多的网络，允许你访问客户机静态的IP，或者将客户机桥接到一个已经存在的网络上。
详情见[networking](https://www.vagrantup.com/docs/networking/)

## 结束方式

如果你暂时不用这个虚拟机了，有三种处理方式。

1. 挂起命令`vagrant suspend`，会保存当前运行状态，不过需要更多磁盘空间
2. 关机命令`vagrant halt`，相对于正常的虚拟机关机
3. 销毁命令`vagrant destroy`，清掉所有的虚拟机资源和磁盘空间

## 重新构建

任何时候只需运行`$ vagrant up`命令即可重新构建你的环境

## FAQ

ubuntu16.04里面，ssh连接超时的解决方案：

```bash
vboxmanage modifyvm winstore_node001 --cableconnected1 on
```

对应在Vagrantfile里面的修正：

```bash
vb.customize ['modifyvm', :id, "--cableconnected1" ,"on"]
```

## 接下来

恭喜你已经入门Vagrant，还有很多好玩的主题呢：

* [虚拟机的配置](https://www.vagrantup.com/docs/virtualbox/)
* [插件](https://www.vagrantup.com/docs/plugins/)
* [自定义同步文件夹](https://www.vagrantup.com/docs/synced-folders/)
* [使用Ansible、Chef作为Provision](https://www.vagrantup.com/docs/provisioning/)