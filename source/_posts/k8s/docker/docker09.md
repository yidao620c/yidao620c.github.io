---
title: Docker每天学一点09 - 多主机管理
date: 2019-03-12 19:15:36
toc: true
categories: [ k8s ]
tags: [ docker ]
---

前面的实验都是在一个host中，而真实环境中肯定会有多个主机。
容器在这些 host 中启动、运行、停止和销毁，相关容器会通过网络相互通信，无论它们是否位于相同的 host。

对于这样一个 multi-host 环境，我们将如何高效地进行管理呢？我们面临的第一个问题是：为所有的 host 安装和配置 docker。
如果一个个去安装肯定很麻烦又容易出错，手工方式效率低且不容易保证一致性，针对这个问题，docker 给出的解决方案是 Docker
Machine。
<!-- more -->

## 安装 Docker Machine

用 Docker Machine 可以批量安装和配置 docker host，这个 host 可以是本地的虚拟机、物理机，也可以是公有云中的云主机。

安装命令：

```bash
curl -L https://github.com/docker/machine/releases/download/v0.16.1/docker-machine-Linux-x86_64 >/tmp/docker-machine &&
chmod +x /tmp/docker-machine &&
sudo cp /tmp/docker-machine /usr/local/bin/docker-machine
```

下载的执行文件被放到 `/usr/local/bin` 中，执行`ocker-mahine version` 验证命令是否可用：

```bash
[root@ecs-hw01 ~]# docker-machine version
docker-machine version 0.16.1, build cce350d7
```

## 配置自动补全

为了得到更好的体验，我们可以安装 bash completion script，这样在 bash 能够通过 tab 键补全 docker-mahine 的子命令和参数。

先安装bash-completion：`yum install bash-completion -y`

然后执行下面脚本下载三个bash文件，复制到`/etc/bash_completion.d/`中：

```bash
scripts=( docker-machine-prompt.bash docker-machine-wrapper.bash docker-machine.bash ); 
for i in "${scripts[@]}"; do sudo wget https://raw.githubusercontent.com/docker/machine/v0.16.1/contrib/completion/bash/${i} -P /etc/bash_completion.d; done
```

启用该功能。 在~/.bashrc中末尾添加：

```bash
source /etc/bash_completion.d/docker-machine-prompt.bash
source /etc/bash_completion.d/docker-machine-wrapper.bash
source /etc/bash_completion.d/docker-machine.bash
PS1='[\u@\h \W$(__docker_machine_ps1)]\$ '
```

重新退出再登录一下bash即可。

## 创建 Machine

对于 Docker Machine 来说，术语 Machine 就是运行 docker daemon 的主机。"创建 Machine" 指的就是在 host 上安装和部署 docker。
先执行 `docker-machine ls` 查看一下当前的 machine：

```bash
[root@ecs-hw01 ~]# docker-machine ls
NAME   ACTIVE   DRIVER   STATE   URL   SWARM   DOCKER   ERRORS
```

当前还没有 machine，接下来我们创建第一个 machine： host1 - 192.168.1.21

创建 machine 要求能够无密码登录远程主机，所以需要先通过如下命令将 ssh key 拷贝到 192.168.1.21：

```bash
ssh-copy-id 192.168.1.21
```

一切准备就绪，执行 `docker-machine create` 命令创建 host1：

```bash
docker-machine create --driver generic --generic-ip-address=192.168.2.21 host1
```

因为我们是往普通的 Linux 中部署 docker，所以使用 generic driver，
其他 driver 可以参考文档 <https://docs.docker.com/machine/drivers/>

`--generic-ip-address` 指定目标系统的 IP，并命名为 host1。

过程如下：

1. 通过 ssh 登录到远程主机
1. 安装 docker
1. 拷贝证书
1. 配置 docker daemon
1. 启动 docker

再次执行 `docker-machine ls`可以看到host1了。登录远程主机我们也看到docker已经安装完成，并且 hostname 已经设置为 host1。
安装同样的方式可以安装host2。

## 管理 Machine

用 docker-machine 创建 machine 的过程很简洁，非常适合多主机环境。除此之外，
Docker Machine 也提供了一些子命令方便对 machine 进行管理。其中最常用的就是无需登录到 machine 就能执行 docker 相关操作。

我们前面学过，要执行远程 docker 命令我们需要通过 -H 指定目标主机的连接字符串，比如：

```bash
docker -H tcp://192.168.12.100:2376 ps
```

Docker Machine 则让这个过程更简单。`docker-machine env host1`显示访问 host1 需要的所有环境变量：

根据提示，执行 `eval $(docker-machine env host1)` 切换到host1中。

下面再介绍几个有用的 docker-machine 子命令

`docker-machine upgrade` 更新 machine 的 docker 到最新版本，可以批量执行：

```bash
docker-machine upgrade host1 host2
```

不要执行stop/start/restart之类，这是对操作系统的命令。

`docker-machine scp` 可以在不同 machine 之间拷贝文件，比如：

```bash
docker-machine scp host1:/tmp/a host2:/tmp/b
```

个人感觉，除了安装docker外，貌似没啥太大用出去，我还是觉得还不如去每台机器上面安装。


