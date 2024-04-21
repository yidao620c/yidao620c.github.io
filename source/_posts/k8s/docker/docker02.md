---
title: Docker每天学一点02 - 架构详解
date: 2019-03-02 09:21:56
toc: true
categories: [ k8s ]
tags: [ docker ]
---

Docker 的核心组件包括：

1. Docker 客户端 - Client
2. Docker 服务器 - Docker daemon
3. Docker 镜像 - Image
4. Docker 注册中心 - Registry
5. Docker 容器 - Container（镜像的运行实例）

Docker 架构图如下：

![](https://xnstatic-1253397658.file.myqcloud.com/docker04.png)

Docker 采用的是 Client/Server 架构。客户端向服务器发送请求，服务器负责构建、运行和分发容器。
客户端和服务器可以运行在同一个 Host 上，客户端也可以通过 socket 或 REST API 与远程的服务器通信。
<!-- more -->

## Docker 客户端

最常用的 Docker 客户端是 docker 命令。通过 docker 我们可以方便地在 Host 上构建和运行容器。

除了 docker 命令行工具，用户也可以通过 REST API 与服务器通信。

## Docker 服务器

Docker daemon 是服务器组件，以 Linux 后台服务的方式运行：

![](https://xnstatic-1253397658.file.myqcloud.com/docker05.png)

Docker daemon 运行在 Docker host 上，负责创建、运行、监控容器，构建、存储镜像。

## Docker 镜像

可将 Docker 镜像看着只读模板，通过它可以创建 Docker 容器。

例如某个镜像可能包含一个 Ubuntu 操作系统、一个 Apache HTTP Server 以及用户开发的 Web 应用。

镜像有多种生成方法：

1. 可以从无到有开始创建镜像
2. 也可以下载并使用别人创建好的现成的镜像
3. 还可以在现有镜像上创建新的镜像

## Docker 容器

Docker 容器就是 Docker 镜像的运行实例。

用户可以通过 CLI（docker）或是 API 启动、停止、移动或删除容器。
可以这么认为，对于应用软件，镜像是软件生命周期的构建和打包阶段，而容器则是启动和运行阶段。

## Registry

Registry 是存放 Docker 镜像的仓库，Registry 分私有和公有两种。

Docker Hub（https://hub.docker.com/） 是默认的 Registry，由 Docker 公司维护，上面有数以万计的镜像，用户可以自由下载和使用。

出于对速度或安全的考虑，用户也可以创建自己的私有 Registry。后面我们会学习如何搭建私有 Registry。

1. docker pull 命令可以从 Registry 下载镜像。
2. docker run 命令则是先下载镜像（如果本地没有），然后再启动容器。

## 容器启动过程

通过之前的运行测试容器的例子来演示一下各个组件协作过程：

```
docker run -d -p 8020:80 httpd
```

1. Docker 客户端执行 docker run 命令。
2. Docker daemon 发现本地没有 httpd 镜像。
3. daemon 从 Docker Hub 下载镜像。
4. 下载完成，镜像 httpd 被保存到本地。
5. Docker daemon 启动容器。

`docker images` 可以查看到 httpd 已经下载到本地：

![](https://xnstatic-1253397658.file.myqcloud.com/docker06.png)

`docker ps` 或者 `docker container ls` 显示容器正在运行：

![](https://xnstatic-1253397658.file.myqcloud.com/docker07.png)

## 小结

Docker 借鉴了集装箱的概念。标准集装箱将货物运往世界各地，Docker 将这个模型运用到自己的设计哲学中，
唯一不同的是：集装箱运输货物，而 Docker 运输软件。

每个容器都有一个软件镜像，相当于集装箱中的货物。容器可以被创建、启动、关闭和销毁。和集装箱一样，
Docker 在执行这些操作时，并不关心容器里到底装的什么，它不管里面是 Web Server，还是 Database。

用户不需要关心容器最终会在哪里运行，因为哪里都可以运行。

开发人员可以在笔记本上构建镜像并上传到 Registry，然后 QA 人员将镜像下载到物理或虚拟机做测试，最终容器会部署到生产环境。

使用 Docker 以及容器技术，我们可以快速构建一个应用服务器、一个消息中间件、一个数据库、一个持续集成环境。
因为 Docker Hub 上有我们能想到的几乎所有的镜像。

如果你是一个运维人员，想研究负载均衡软件 HAProxy，只需要执行docker run haproxy，无需繁琐的手工安装和配置既可以直接进入实战。

如果你是一个开放人员，想学习怎么用 django 开发 Python Web 应用，执行 docker run django，
在容器里随便折腾吧，不用担心会搞乱 Host 的环境。

不夸张的说：容器大大提升了 IT 人员的幸福指数。

还在犹豫啥，Docker赶紧搞起来。^_^
