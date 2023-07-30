---
title: Docker每天学一点03 - 镜像
date: 2019-03-03 08:26:12
toc: true
categories: [ k8s ]
tags: [ docker ]
---

镜像是 Docker 容器的基石，容器是镜像的运行实例，有了镜像才能启动容器。

如果我们想要创建自己的镜像，肯定要先了解镜像的内部结构。

hello-world 是 Docker 官方提供的一个镜像，通常用来验证 Docker 是否安装成功。
我先从这个最小镜像开始说下，之前已经把它下载下来了。

hello-world 的 Dockerfile 内容如下：

```
FROM scratch    # 此镜像是从白手起家，从 0 开始构建。
COPY hello /    # 将文件"hello"复制到镜像的根目录。
CMD ["/hello"]  # 容器启动时，执行 /hello
```

镜像 hello-world 中就只有一个可执行文件 "hello"，其功能就是打印出 "Hello from Docker ......" 等信息。
<!-- more -->

/hello 就是文件系统的全部内容，连最基本的 /bin，/usr, /lib, /dev 都没有。

hello-world 虽然是一个完整的镜像，但它并没有什么实际用途。通常来说，我们希望镜像能提供一个基本的操作系统环境，
用户可以根据需要安装和配置软件。这样的镜像我们称作 base 镜像。

## base镜像

实际上，base镜像就是这样的：

1. 不依赖其他镜像，从 scratch 构建。
2. 其他镜像可以之为基础进行扩展。

所以，能称作 base 镜像的通常都是各种 Linux 发行版的 Docker 镜像，比如 Ubuntu, Debian, CentOS 等。

我们下载一个名字叫centos的base镜像：

```
docker pull centos
```

查看镜像信息：

```
docker images centos
```

![](https://xnstatic-1253397658.file.myqcloud.com/docker11.png)

镜像大小不到 200MB。

等一下！一个 CentOS 才 200MB ？平时我们安装一个 CentOS 至少都有几个 GB，怎么可能才 200MB !

Linux 操作系统由内核空间和用户空间组成。

内核空间是 kernel，Linux 刚启动时会加载 bootfs 文件系统，之后 bootfs 会被卸载掉。

用户空间的文件系统是 rootfs，包含我们熟悉的 /dev, /proc, /bin 等目录。

对于 base 镜像来说，底层直接用 Host 的 kernel，自己只需要提供 rootfs 就行了。

而对于一个精简的 OS，rootfs 可以很小，只需要包括最基本的命令、工具和程序库就可以了。
相比其他 Linux 发行版，CentOS 的 rootfs 已经算臃肿的了，alpine 还不到 10MB。

## 镜像的分层结构

Docker 支持通过扩展现有镜像，创建新的镜像。

实际上，Docker Hub 中 99% 的镜像都是通过在 base 镜像中安装和配置需要的软件构建出来的。
比如我们现在构建一个新的镜像，Dockerfile 如下：

![](https://xnstatic-1253397658.file.myqcloud.com/docker08.png)

1. 新镜像不再是从 scratch 开始，而是直接在 Debian base 镜像上构建。
2. 安装 emacs 编辑器。
3. 安装 apache2。
4. 容器启动时运行 bash。

构建过程如下图所示：

![](https://xnstatic-1253397658.file.myqcloud.com/docker09.png)

可以看到，新镜像是从 base 镜像一层一层叠加生成的。每安装一个软件，就在现有镜像的基础上增加一层。

## 可写的容器层

当容器启动时，一个新的可写层被加载到镜像的顶部。
这一层通常被称作"容器层"，"容器层"之下的都叫"镜像层"。

![](https://xnstatic-1253397658.file.myqcloud.com/docker11.png)

所有对容器的改动 - 无论添加、删除、还是修改文件都只会发生在容器层中。
只有容器层是可写的，容器层下面的所有镜像层都是只读的。

## Dockerfile构建镜像

Dockerfile 是一个文本文件，记录了镜像构建的所有步骤。

我们就在上面下载的centos这个基础镜像上面构建一个安装vim软件的新的镜像。

先到/root目录下面，新增一个Dockerfile文件，内容如下：

```
FROM centos
RUN yum update -y && yum install -y vim
```

然后执行构建命令：

```
docker build -t centos-with-vim .
```

注意到最后一个"."，表示构建的context为当前目录。我只截取最后的输出：

![](https://xnstatic-1253397658.file.myqcloud.com/docker12.png)

安装成功后，将容器保存为镜像，其 ID 为 23c68986bc8e。

通过 docker images 查看镜像信息：

![](https://xnstatic-1253397658.file.myqcloud.com/docker13.png)

镜像 ID 为 23c68986bc8e，与构建时的输出一致。

## 查看镜像分层结构

centos-with-vim这个镜像是在基础镜像centos的顶部添加了一层新的镜像而得到。
这一点我们可以通过 docker history 命令验证：

![](https://xnstatic-1253397658.file.myqcloud.com/docker14.png)

可以看到与 centos 镜像相比，确实只是多了顶部的一层23c68986bc8e

## 镜像的缓存特性

Docker 会缓存已有镜像的镜像层，构建新镜像时，如果某镜像层已经存在，就直接使用，无需重新创建。

## Dockerfile调试

通过 Dockerfile 构建镜像的过程：

1. 从 base 镜像运行一个容器。
2. 执行一条指令，对容器做修改。
3. 执行类似 docker commit 的操作，生成一个新的镜像层。
4. Docker 再基于刚刚提交的镜像运行一个新容器。
5. 重复 2-4 步，直到 Dockerfile 中的所有指令执行完毕。

从这个过程可以看出，如果 Dockerfile 由于某种原因执行到某个指令失败了，
我们也将能够得到前一个指令成功执行构建出的镜像，这对调试 Dockerfile 非常有帮助。
我们可以运行最新的这个镜像定位指令失败的原因。

比如，运行`docker build`总共3步，前面两步都成功了，第3步失败。第2步构建的docker id 为22d31cc52b3e。

Dockerfile 在执行第三步 RUN 指令时失败。我们可以利用第二步创建的镜像 22d31cc52b3e 进行调试，
方式是通过 docker run -it 启动镜像的一个容器。

```
docker run -it 22d31cc52b3e
```

然后就不用重复之前的正确构建步骤，直接找到问题所在了。

## 镜像操作命令

仓库中镜像查询：

```
docker search image #查询中央仓库中的镜像
curl -XGET http://192.168.1.8:5000/v2/nginx/tags/list #查询私有仓库中的镜像
```

本地镜像查询：

```
docker images image
docker images -a
```

拉取镜像：

```
docker pull image:tag #拉取中央仓库中镜像
docker pull localhost:5000/yidao620/httpd:v1 #拉取私有仓库中镜像 
```

删除本地镜像：

```
docker rmi image
docker rmi `docker images -a -q`
```

## 删除私有仓库镜像

对于私有仓库的镜像，我们无法直接删除。当随着镜像的越来越多，会导致磁盘空间越来越小

阅读registry v2的http API后发现删除镜像需要调用几个API

* 获取image的digest
* 删除镜像的manifests

下面是删除私有registry的镜像脚本clean.sh

```
Usage：bash clean.sh your-image-name
```

需要安装jq，jq是终端解析json输出的利器

```bash
#!/usr/bin/env bash

ACCEPT_HEADER="Accept: application/vnd.docker.distribution.manifest.v2+json"
DOCKER_REGISTRY="your-registry:35000/v2"
AUTH="-uusername:password"
REPOSITORY=$1

TAGS=`curl --silent -X GET -k $AUTH $DOCKER_REGISTRY/$REPOSITORY/tags/list | jq -r '."tags"[]'`
echo image $REPOSITORY has tags: $TAGS

for TAG in ${TAGS[@]}
do
	echo  "i am going to delete $REPOSITORY:$TAG"
	digest_value=`curl -X GET -k --head --silent -H "Accept: application/vnd.docker.distribution.manifest.v2+json" $AUTH $DOCKER_REGISTRY/$REPOSITORY/manifests/$TAG 2>&1 | grep Docker-Content-Digest | awk '{print $2}'`

	digest_url="$DOCKER_REGISTRY/$REPOSITORY/manifests/$digest_value"
	echo $digest_url
	URL=${digest_url%$'\r'}
	curl -X DELETE -k -H  "Accept: application/vnd.docker.distribution.manifest.v2+json" $AUTH  $URL
done

#docker exec -it registry bin/registry garbage-collect /etc/docker/registry/config.yml

#REPOSITORY_PATH="/var/lib/registry/docker/registry/v2/repositories"
#docker exec -it registry rm -rf $REPOSITORY_PATH/$REPOSITORY
```

脚本里面注释的几行需要解释一下：

* gc回收，我们删除image的mainfests的时候，仓库并不会删除image，只有当你调用GC回收的时候，才会删除。这个可以用crontab定时执行。
* 删除具体的镜像目录。（当我们用GC回收的时候，其实只是把镜像名字从仓库里面去掉了。但是实际上磁盘还是存在这个镜像的。这个只能是通过rm删除

