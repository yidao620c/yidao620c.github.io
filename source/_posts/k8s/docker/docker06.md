---
title: Docker每天学一点06 - 运行容器
date: 2019-03-09 09:16:33
toc: true
categories: [ k8s ]
tags: [ docker ]
---

这一篇学习容器的各种操作，容器的状态之间如何转换，以及实现容器的底层技术。

运行容器

```bash
docker run -d --name "node001" httpd
```

查看当前正在运行的容器

```bash
docker ps
docker container ls
```

<!-- more -->

返回结果：

```bash
[root@VM_22_2_centos ~]# docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                    NAMES
2d33b4ef85c4        registry:2          "/entrypoint.sh /e..."   7 hours ago         Up 7 hours          0.0.0.0:5000->5000/tcp   vibrant_mestorf
ef4e11f77711        my-image            "/bin/bash"              2 days ago          Up 2 days                                    cranky_yalow
3d5ea3e73344        httpd               "httpd-foreground"       4 days ago          Up 4 days           0.0.0.0:8020->80/tcp     boring_bardeen
```

加一个`-a`选项可把所有容器都列出来：

```bash
docker ps -a
```

让容器长期运行，可以通过执行一个长期运行的命令来保持容器的运行状态，然后使用`-d`选项让它以后台方式启动容器。

```bash
docker run -d httpd
```

`CONTAINER ID` 是容器的 "短ID"，前面启动容器时返回的是 "长ID"。短ID是长ID的前12个字符。

`NAMES` 字段显示容器的名字，在启动容器时可以通过 `--name` 参数显示地为容器命名，
如果不指定，docker 会自动为容器分配名字。

对于容器的后续操作，我们需要通过 "长ID"、"短ID" 或者 "名称" 来指定要操作的容器。比如下面停止一个容器：

```bash
docker stop 3d5ea3e73344
```

查看容器启动时候执行的所有命令，`docker history` 会显示镜像的构建历史，也就是 Dockerfile 的执行过程。
注意前面讲的容器是一个层次结构，从底部向上面一层一层构建的：

```bash
docker history httpd
```

## 如果让一个容器长期运行

默认情况下，容器中运行的进程如果没有前台进程了或者所有的前台进程都运行完只剩下后台进程，则容器会退出。
因为Docker容器仅在它的1号进程（PID为1）运行时，会保持运行。如果1号进程退出了，Docker容器也就退出了。

有几种办法可以让容器长期运行：

1. 使用runit或supervisord这类进程管理器来运行进程
2. 通过指定一些运行参数将后台服务以前台进程方式运行，比如`nginx -g 'daemon off;'`，`/usr/sbin/apache2 -D FOREGROUND`
3. 如果运行自己的脚本，可最后添加 `exec /bin/bash`，启动容器时候`docker run -dit image`即可

## 进入容器方法

我们经常需要进到容器里去做一些工作，比如查看日志、调试、启动其他进程等。有两种方法进入容器：attach 和 exec。

attach 方式：

```bash
docker attach long_id
```

exec方式：

```bash
docker exec -it <container> bash|sh
```

attach 与 exec 主要区别如下:

1. attach直接进入容器启动命令的终端，不会启动新的进程。
2. exec 则是在容器中打开新的终端，并且可以启动新的进程。
3. 如果想直接在终端中查看启动命令的输出，用attach；其他情况使用 exec。

如果只是为了查看启动命令的输出，可以使用 docker logs 命令：

```bash
docker logs -f 3d5ea3e73344
```

`-f` 的作用与 `tail -f` 类似，能够持续打印输出。

**查看容器进程**

```bash
docker top busybox
```

**查看容器端口映射**

```bash
docker port busybox
```

## 运行容器最佳实践

按用途容器大致可分为两类：服务类容器和工具类的容器。

1\. 服务类容器以 daemon 的形式运行，对外提供服务。比如 web server，数据库等。
通过 -d 以后台方式启动这类容器是非常合适的。如果要排查问题，可以通过 `exec -it` 进入容器。

2\. 工具类容器通常给能我们提供一个临时的工作环境，通常以 `run -it` 方式运行，比如：

```bash
docker run -it busybox

/# wget www.baidu.com
/# exit
```

运行 busybox，`run -it` 的作用是在容器启动后就直接进入。我们这里通过 wget 验证了在容器中访问 internet 的能力。
执行 `exit` 退出终端，同时容器停止。

工具类容器多使用基础镜像，例如 busybox、debian、ubuntu 等。

## 容器常用操作

启动、停止、重启容器：

```bash
docker stop boring_bardeen
docker start boring_bardeen
docker restart boring_bardeen
```

容器可能会因某种错误而停止运行。对于服务类容器，我们通常希望在这种情况下容器能够自动重启。
启动容器时设置 --restart 就可以达到这个效果。

```bash
docker run -d --restart=always httpd
```

暂停容器：

```bash
docker pause boring_bardeen
```

![](https://xnstatic-1253397658.file.myqcloud.com/docker20.png)

处于暂停状态的容器不会占用 CPU 资源，直到通过 docker unpause 恢复运行：

```bash
docker unpause boring_bardeen
```

![](https://xnstatic-1253397658.file.myqcloud.com/docker21.png)

删除容器

使用 docker 一段时间后，host 上可能会有大量已经退出了的容器，
状态为Exited，这些容器依然会占用 host 的文件系统资源，如果确认不会再重启此类容器，
可以通过 docker rm 删除：

```bash
docker ps -a |grep "Exited"
docker rm de97841fbc9e
```

docker rm 一次可以指定多个容器，如果希望批量删除所有已经退出的容器，可以执行如下命令：

```bash
docker rm -v $(docker ps -aq -f status=exited)
```

注意：`docker rm` 是删除容器，而 `docker rmi` 是删除镜像。

## 容器状态

1. `docker create` 创建的容器处于 Created 状态
2. `docker start` 将以后台方式启动容器，容器状态处于 Running 状态。
3. 实际上，`docker run` 命令是`docker create` 和 `docker start` 的组合

## 持久化容器

前面我讲过了如何利用Dockerfile再其他已有镜像基础上创建新的镜像了。
这里再讲一下如何将运行中的容器进行持久化。

### docker commit

可以通过 `docker commit` 将一个运行中的容器变成一个新的镜像，命令如下：

```bash
docker commit <containner-id> image-name
```

### 导出(Export)

`docker export` 命令用于持久化容器（不是镜像）：

```bash
sudo docker export <CONTAINER ID> > /home/export.tar
```

导入容器得到新的镜像：

```bash
# 导入export.tar文件
cat /home/export.tar | sudo docker import - busybox-1-export:latest

# 查看镜像
sudo docker images
```

### 保存(Save)

Save命令用于持久化镜像（不是容器）：

```bash
sudo docker save image-name > /home/save.tar
```

导入镜像得到新的镜像：

```bash
 # 导入save.tar文件
docker load < /home/save.tar

# 查看镜像
sudo docker images
```

### 导出和保存的差别

导出后再导入(exported-imported)的镜像会丢失所有的历史，而保存后再加载（saveed-loaded）的镜像没有丢失历史和层(layer)。
这意味着使用导出后再导入的方式，你将无法回滚到之前的层(layer)，同时，使用保存后再加载的方式持久化整个镜像，
就可以做到层回滚（可以执行docker tag <LAYER ID> <IMAGE NAME>来回滚之前的层）。

```bash
# 显示镜像的所有层(layer)
docker history image-name
```

## 内存限额

与操作系统类似，容器可使用的内存包括两部分：物理内存和 swap。Docker 通过下面两组参数来控制容器内存的使用量：

1. -m 或 --memory：设置内存的使用限额，例如 100M, 2G。
2. --memory-swap：设置 `内存+swap` 的使用限额。

当我们执行如下命令：

```bash
docker run -m 200M --memory-swap=300M httpd
```

其含义是允许该容器最多使用 200M 的内存和 100M 的 swap。默认情况下，
上面两组参数为 -1，即对容器内存和 swap 的使用没有限制。

如果在启动容器时只指定 -m 而不指定 --memory-swap，那么 --memory-swap 默认为 -m 的两倍。

## CPU限制

默认设置下，所有容器可以平等地使用 host CPU 资源并且没有限制。

Docker 可以通过 `-c` 或 `--cpu-shares` 设置容器使用 CPU 的权重。如果不指定，默认值为 1024。

与内存限额不同，通过 -c 设置的 cpu share 并不是 CPU 资源的绝对数量，而是一个相对的权重值。
某个容器最终能分配到的 CPU 资源取决于它的 cpu share 占所有容器 cpu share 总和的比例。

换句话说：通过 cpu share 可以设置容器使用 CPU 的优先级。

比如在 host 中启动了两个容器：

```bash
docker run --name "container_A" -c 1024 centos
docker run --name "container_B" -c 512 centos
```

container_A 的 cpu share 1024，是 container_B 的两倍。
当两个容器都需要 CPU 资源时，container_A 可以得到的 CPU 是 container_B 的两倍。

## 磁盘IO限制

Block IO 是另一种可以限制容器使用的资源。Block IO 指的是磁盘的读写，
docker 可通过设置权重、限制 bps 和 iops 的方式控制容器读写磁盘的带宽，下面分别讨论。

注：目前 Block IO 限额只对 direct IO（不使用文件缓存）有效。

默认情况下，所有容器能平等地读写磁盘，可以通过设置 `--blkio-weight` 参数来改变容器 block IO 的优先级。

`--blkio-weight` 与 `--cpu-shares` 类似，设置的是相对权重值，默认为 500。
在下面的例子中，container_A 读写磁盘的带宽是 container_B 的两倍：

```bash
docker run -it --name container_A --blkio-weight 600 centos   
docker run -it --name container_B --blkio-weight 300 centos
```

**限制 bps 和 iops**

* bps 是 byte per second，每秒读写的数据量。
* iops 是 io per second，每秒 IO 的次数。

可通过以下参数控制容器的 bps 和 iops：

1. --device-read-bps，限制读某个设备的 bps。
2. --device-write-bps，限制写某个设备的 bps。
3. --device-read-iops，限制读某个设备的 iops。
4. --device-write-iops，限制写某个设备的 iops。

下面这个例子限制容器写 /dev/sda 的速率为 30 MB/s ：

```
docker run -it --device-write-bps /dev/sda:30MB centos
```

## 容器底层技术

cgroup 和 namespace 是最重要的两种技术。cgroup 实现资源限额， namespace 实现资源隔离。

**cgroup**

cgroup 全称 Control Group。Linux 操作系统通过 cgroup 可以设置进程使用 CPU、内存 和 IO 资源的限额。
前面我们看到的 `--cpu-shares`、`-m`、`--device-write-bps` 实际上就是在配置 cgroup。

**namespace**

在每个容器中，我们都可以看到文件系统，网卡等资源，这些资源看上去是容器自己的。
拿网卡来说，每个容器都会认为自己有一块独立的网卡，即使 host 上只有一块物理网卡。这种方式非常好，它使得容器更像一个独立的计算机。

Linux 实现这种方式的技术是 namespace。namespace 管理着 host 中全局唯一的资源，并可以让每个容器都觉得只有自己在使用它。
换句话说，namespace 实现了容器间资源的隔离。

Linux 使用了六种 namespace，分别对应六种资源：Mount、UTS、IPC、PID、Network 和 User。

### Mount namespace

Mount namespace 让容器看上去拥有整个文件系统。

容器有自己的 / 目录，可以执行 mount 和 umount 命令。当然我们知道这些操作只在当前容器中生效，不会影响到 host 和其他容器。

### UTS namespace

UTS namespace 让容器有自己的 hostname。默认情况下，容器的 hostname 是它的短ID，可以通过 -h 或 --hostname 参数设置。

```
docker run -h xnhost -it centos
```

### IPC namespace

IPC namespace 让容器拥有自己的共享内存和信号量（semaphore）来实现进程间通信，而不会与 host 和其他容器的 IPC 混在一起。

### PID namespace

所有容器的进程都挂在 dockerd 进程下，同时也可以看到容器自己的子进程。
如果我们进入到某个容器，ps 就只能看到自己的进程了。
容器拥有自己独立的一套 PID，这就是 PID namespace 提供的功能。

### Network namespace

Network namespace 让容器拥有自己独立的网卡、IP、路由等资源。我们会在后面网络章节详细讨论。

### User namespace

User namespace 让容器能够管理自己的用户，host 不能看到容器中创建的用户。


