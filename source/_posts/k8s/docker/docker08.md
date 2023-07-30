---
title: Docker每天学一点08 - 存储卷
date: 2019-03-11 10:37:55
toc: true
categories: [ k8s ]
tags: [ docker ]
---

Docker 为容器提供了两种存放数据的资源：

1. 由 storage driver 管理的镜像层和容器层。
2. Data Volume

接下来分别介绍下这两种存储方式。
<!-- more -->

## storage driver

在镜像章节我们学习到 Docker 镜像的分层结构：

![](https://xnstatic-1253397658.file.myqcloud.com/docker33.png)

容器由最上面一个可写的容器层，以及若干只读的镜像层组成，容器的数据就存放在这些层中。这样的分层结构最大的特性是
Copy-on-Write：

1. 新数据会直接存放在最上面的容器层。
1. 修改现有数据会先从镜像层将数据复制到容器层，修改后的数据直接保存在容器层中，镜像层保持不变。
1. 如果多个层中有命名相同的文件，用户只能看到最上面那层中的文件。

分层结构使镜像和容器的创建、共享以及分发变得非常高效，而这些都要归功于 Docker storage driver。
正是 storage driver 实现了多层数据的堆叠并为用户提供一个单一的合并之后的统一视图。

Docker 安装时会根据当前系统的配置选择默认的 driver。默认 driver 具有最好的稳定性，因为默认 driver 在发行版上经过了严格的测试。

运行docker info查看 CentOS 7 的默认 driver：

```
[root@VM_22_2_centos docker]# docker info
Containers: 3
 Running: 3
 Paused: 0
 Stopped: 0
Images: 17
Server Version: 18.03.0-ce
Storage Driver: devicemapper
```

可见CentOS 7的默认driver是devicemapper

对于某些容器，直接将数据放在由 storage driver 维护的层中是很好的选择，比如那些无状态的应用。
无状态意味着容器没有需要持久化的数据，随时可以从镜像直接创建。

比如 busybox，它是一个工具箱，我们启动 busybox 是为了执行诸如 wget，ping 之类的命令，不需要保存数据供以后使用，
使用完直接退出，容器删除时存放在容器层中的工作数据也一起被删除，这没问题，下次再启动新容器即可。

对于大部分容器而言，都有持久化数据的需求，容器启动时需要加载已有的数据，容器销毁时希望保留产生的新数据，也就是说，这类容器是有状态的。
这就要用到 Docker 的另一种存储机制：Data Volume

## Data Volume

Data Volume 本质上是 Docker Host 文件系统中的目录或文件，有以下几个特点：

1. Data Volume 是目录或文件，而非没有格式化的磁盘（块设备）。
1. 容器可以读写 volume 中的数据。
1. volume 数据可以被永久的保存，即使使用它的容器已经销毁。

docker 提供了两种类型的 volume：bind mount 和 docker managed volume。

**bind mount**

bind mount 是将 host 上已存在的目录或文件 mount 到容器。

例如 docker host 上有目录 /opt/htdocs ：

```
[root@VM_22_2_centos docker]# cat /opt/htdocs/index.html 
<h1>Hi, Girl.</h1>
```

然后通过 -v 参数将这个目录mount到httpd容器：

```
[root@VM_22_2_centos docker]# docker run -d -p 8080:80 -v /opt/htdocs:/usr/local/apache2/htdocs httpd
9ee8a32d718345e2e60496eb3ad93d95a7a933646cb0fd8c65b3af4fd71853ba
```

注意，在主机上面是没有容器的目录的`/usr/local/apache2/htdocs`

```
[root@VM_22_2_centos docker]# ll /usr/local/apache2/htdocs
ls: cannot access /usr/local/apache2/htdocs: No such file or directory
```

-v 的格式为 `<host path>:<container path>`

`/usr/local/apache2/htdocs` 就是 apache server 存放静态文件的地方。由于之前在容器中 `/usr/local/apache2/htdocs` 已经存在，
原有数据会被隐藏起来，取而代之的是 host上目录 `/opt/htdocs/` 中的数据，这与 linux mount 命令的行为是一致的。

```
[root@VM_22_2_centos docker]# curl localhost:8080
<h1>Hi, Girl.</h1>
[root@VM_22_2_centos docker]# 
```

curl 显示当前主页确实是 /opt/htdocs/index.html 中的内容。更新一下，看是否能生效：

```
[root@VM_22_2_centos docker]# echo "updated index page" > /opt/htdocs/index.html 
[root@VM_22_2_centos docker]# curl localhost:8080
updated index page
[root@VM_22_2_centos docker]#
```

host 中的修改确实生效了，bind mount 可以让 host 与容器共享数据。这在管理上是非常方便的。

除了 bind mount 目录，还可以单独指定一个文件。

```
docker run -d -p 80:80 -v ~/htdocs/index.html:/usr/local/apache2/htdocs/new_index.html httpd
```

使用 bind mount 单个文件的场景是：只需要向容器添加文件，不希望覆盖整个目录。
在上面的例子中，我们将 html 文件加到 apache 中，同时也保留了容器原有的数据。
使用单一文件有一点要注意：host 中的源文件必须要存在，不然会当作一个新目录 bind mount 给容器。

mount point 有很多应用场景，比如我们可以将源代码目录 mount 到容器中，在 host 中修改代码就能看到应用的实时效果。
再比如将 mysql 容器的数据放在 bind mount 里，这样 host 可以方便地备份和迁移数据。

bind mount 的使用直观高效，易于理解，但它也有不足的地方：bind mount 需要指定 host 文件系统的特定路径，这就限制了容器的可移植性，
当需要将容器迁移到其他 host，而该 host 没有要 mount 的数据或者数据不在相同的路径时，操作会失败。

备注：这种方式的共享用来主机和容器共享，非常适合开发阶段。

移植性更好的方式是 docker managed volume，就是接下来我要讲解的。

**docker managed volume**

docker managed volume 与 bind mount 在使用上的最大区别是不需要指定 mount 源，指明 mount point 就行了。还是以 httpd 容器为例：

```
docker run -d -p 8080:80 -v /usr/local/apache2/htdocs httpd
```

每当容器申请 `docker managed volume` 时，docker 都会在`/var/lib/docker/volumes` 下生成一个目录，这个目录就是 mount 源。
volume 的内容跟容器中原有 `/usr/local/apache2/htdocs`目录里面内容完全一样，这是怎么回事呢？

这是因为：如果 mount point 指向的是已有目录，原有数据会被复制到 volume 中。

此时的 `/usr/local/apache2/htdocs` 已经不再是由 storage driver 管理的层数据了，它已经是一个 data volume。
我们可以像 bind mount 一样对数据进行操作。

docker managed volume 的创建过程：

1. 容器启动时，简单的告诉 docker "我需要一个 volume 存放数据，帮我 mount 到目录 /abc"。
1. docker 在`/var/lib/docker/volumes` 中生成一个随机目录作为 mount 源。
1. 如果容器中的目录 /abc 已经存在，则将其中的数据复制到 mount 源，
1. 将 volume mount 到 /abc

## volume container

volume container（数据卷容器） 是专门为其他容器提供 volume 的容器。它提供的卷可以是 bind mount，也可以是 docker managed
volume。

接下来我创建一个volume container试试看：

```
[root@VM_22_2_centos docker]# docker create --name vc_data \
> -v /opt/htdocs:/usr/local/apache2/htdocs \
> -v /other/useful/tools \
> busybox
f9960d7960fce345c983e77435b38a8044f4805b2ce2682d86e09dc2fbc0e350
```

我们将容器命名为 vc_data（vc 是 volume container 的缩写）。注意这里执行的是 docker create 命令，
这是因为 volume container 的作用只是提供数据，它本身不需要处于运行状态。容器 mount 了两个 volume：

1. bind mount，存放 web server 的静态文件。
1. docker managed volume，存放一些实用工具（当然现在是空的，这里只是做个示例）。

通过 `docker inspect vc_data` 可以查看到这两个 volume

![](https://xnstatic-1253397658.file.myqcloud.com/docker34.png)

其他容器可以通过 `--volumes-from` 使用 vc_data 这个 volume container。下面我启动三个httpd的容器，并共享这个存储容器：

```
[root@VM_22_2_centos docker]# docker run --name web1 -d -p 80 --volumes-from vc_data httpd 
55dca74022ac8409282249dd7603671c4dad21fbfdabe2b7f63a68fd1a872e18
[root@VM_22_2_centos docker]# docker run --name web2 -d -p 80 --volumes-from vc_data httpd 
6408133e7237dc0ea1d1c273b228c0bae5b3a0ccf1035ba3bd6fb3c13216cb14
[root@VM_22_2_centos docker]# docker run --name web3 -d -p 80 --volumes-from vc_data httpd 
86a3b71c8d057fd52a3c31206129c5732352d6f4750b1160b75c82c158534b4f
```

验证一下数据共享的效果：

```
[root@VM_22_2_centos docker]# echo "this content is from a volume containner" > /opt/htdocs/index.html 
[root@VM_22_2_centos docker]# curl 127.0.0.1:32770
this content is from a volume containner
[root@VM_22_2_centos docker]# curl 127.0.0.1:32769
this content is from a volume containner
[root@VM_22_2_centos docker]# curl 127.0.0.1:32768
this content is from a volume containner
```

可见，三个容器已经成功共享了 volume container 中的 volume。

总结一下 volume container 的特点：

1. 与 bind mount 相比，不必为每一个容器指定 host path，所有 path 都在 volume container 中定义好了，
   容器只需与 volume container 关联，实现了容器与 host 的解耦。
1. 使用 volume container 的容器其 mount point 是一致的，有利于配置的规范和标准化，但也带来一定的局限，使用时需要综合考虑。

## data-packed volume container

上面讨论的volume container 的数据归根到底还是在 host 里，有没有办法将数据完全放到 volume container 中，同时又能与其他容器共享呢？

当然可以，通常我们称这种容器为 data-packed volume container。其原理是将数据打包到镜像中，然后通过 docker managed volume 共享。

我们用下面的 Dockfile 构建镜像：

```
FROM busybox:lastest
ADD htdocs /usr/local/apache2/htdocs
VOLUME /usr/local/apache2/htdocs
```

![](https://xnstatic-1253397658.file.myqcloud.com/docker35.png)

ADD 将静态文件目录`htdocs`复制到容器目录 `/usr/local/apache2/htdocs`

VOLUME 的作用与 -v 等效，用来创建 docker managed volume，mount point 为 `/usr/local/apache2/htdocs`，
因为这个目录就是 ADD 添加的目录，所以会将已有数据拷贝到 volume 中。

先删除之前的vc_data，然后用新镜像创建 data-packed volume container：

```
[root@VM_22_2_centos docker]# docker rm vc_data
[root@VM_22_2_centos docker]# docker create --name vc_data datapacked
301cbcff7e654a8923e1ab2ea543d4f3d9e80bdb37e1c821990dc09914585318
[root@VM_22_2_centos docker]# docker run -d -p 80 --volumes-from vc_data httpd
56598537f816e10d6b7a88eb942291761e2eeb4e004615b39cfcf3c326ea06cf
[root@VM_22_2_centos docker]# docker ps
CONTAINER ID        IMAGE               COMMAND                CREATED             STATUS              PORTS                                                                 NAMES
56598537f816        httpd               "httpd-foreground"     8 seconds ago       Up 6 seconds        0.0.0.0:32771->80/tcp                                                 zealous_leakey
[root@VM_22_2_centos docker]# curl 127.0.0.1:32771
this content is from a volume containner
```

容器能够正确读取 volume 中的数据。data-packed volume container 是自包含的，不依赖 host 提供数据，具有很强的移植性，
非常适合**只使用**静态数据的场景，比如应用的配置信息、web server 的静态文件等。对于开发和生产环境中有不同的配置文件的时候很方便。

## volume 生命周期

最后一节讨论如何备份、恢复、迁移和销毁 data volume。

**备份**

因为 volume 实际上是 host 文件系统中的目录和文件，所以 volume 的备份实际上是对文件系统的备份。

**恢复**

volume 的恢复也很简单，如果数据损坏了，直接用之前备份的数据拷贝到value的源目录可以了。

**迁移**

如果我们想使用更新版本的镜像启动容器，这就涉及到数据迁移，方法是：

1. docker stop 当前 Registry 容器。
2. 启动新版本容器并 mount 原有 volume。

```
docker run -d -p 5000:5000 -v /myregistry:/var/lib/registry registry:latest
```

当然，在启用新容器前要确保新版本的默认数据路径是否发生变化。

**销毁**

可以删除不再需要的 volume，但一定要确保知道自己正在做什么，volume 删除后数据是找不回来的。

* docker 不会销毁 bind mount，删除数据的工作只能由 host 负责。
* 对于 docker managed volume，在执行 docker rm 删除容器时可以带上 -v 参数，

docker 会将容器使用到的 volume 一并删除，但前提是没有其他容器 mount 该 volume，目的是保护数据，非常合理。

如果想批量删除孤儿 volume，可以执行：

```
docker volume rm $(docker volume ls -q)
```

这章我们学习的只是单个 docker host 中的存储方案。而跨主机存储也是一个重要的主题，当然也更复杂，我们会在容器进阶技术章节详细讨论。
