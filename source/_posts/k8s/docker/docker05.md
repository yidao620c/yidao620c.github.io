---
title: Docker每天学一点05 - Registry
date: 2019-03-07 12:07:09
toc: true
categories: [ k8s ]
tags: [ docker ]
---

前面已经学会怎样构建镜像了，这一章介绍怎样分发镜像给多个Docker Host使用，
可以通过复制Dockerfile、将镜像上传至公共Registry、搭建私有Registry三种方式。

这里将后面两种方式，怎样使用公共Registry和搭建私有Registry。
<!-- more -->

## 镜像命名

其实就跟maven库管理一样，镜像名字由两部分组成：repository 和 tag：

```
[image name] = [repository]:[tag]
```

如果执行 docker build 时没有指定 tag，会使用默认值 latest。其效果相当于：

```
docker build -t ubuntu-with-vi:latest
```

在实际使用中，我们最好指定版本（也就是标签tag），比如`httpd:2.3`

我们可以通过 docker tag 命令方便地给镜像打 tag

```
docker tag myimage-v1.9.1 myimage:1
docker tag myimage-v1.9.1 myimage:1.9
docker tag myimage-v1.9.1 myimage:1.9.1
docker tag myimage-v1.9.1 myimage:latest
```

当我们发布2.0版本的时候，将latest换成2.0版本：

```
docker tag myimage-v2.0.0 myimage:2
docker tag myimage-v2.0.0 myimage:2.0
docker tag myimage-v2.0.0 myimage:2.0.0
docker tag myimage-v2.0.0 myimage:latest
```

## 使用公共 Registry

保存和分发镜像的最直接方法就是使用 Docker Hub。

Docker Hub 是 Docker 公司维护的公共 Registry。
用户可以将自己的镜像保存到 Docker Hub 免费的 repository 中。
如果不希望别人访问自己的镜像，也可以购买私有 repository。

1\. 首先在 [Docker Hub](https://hub.docker.com/) 上面注册一个账号。

2\. 在 Docker Host 上登录：

```
docker login -u yidao620
```

这里用的是我自己的账号，用户名为 yidao620，输入密码后登录成功。

3\. 修改镜像的 repository 使之与 Docker Hub 账号匹配。

Docker Hub 为了区分不同用户的同名镜像，镜像的 registry 中要包含用户名，完整格式为：[username]/xxx:tag

我们通过 docker tag 命令重命名镜像，我要把之前下载的httpd上传上去。

```
docker tag httpd yidao620/httpd:v1
docker images
```

![](https://xnstatic-1253397658.file.myqcloud.com/docker17.png)

注：Docker 官方自己维护的镜像没有用户名，比如 httpd

4\. 通过 docker push 将镜像上传到 Docker Hub：

```
docker push yidao620/httpd:v1
```

Docker 会上传镜像的每一层。因为 `yidao620/httpd:v1` 这个镜像实际上跟官方的 httpd 镜像一模一样，
Docker Hub 上已经有了全部的镜像层，所以真正上传的数据很少，大概几秒就完成上传了。

5\. 登录 [Docker Hub](https://hub.docker.com/) ,在Public Repository 中就可以看到上传的镜像了。

![](https://xnstatic-1253397658.file.myqcloud.com/docker18.png)

6\. 这个镜像可被其他 Docker host 下载使用了。

```
docker pull yidao620/httpd:v1
```

公共Registry使用很简单，接下来看看怎样搭建本地Registry。

## 搭建本地Registry

处于安全和网络传输原因，通常企业内部不会再公共仓库上面维护的镜像，而是搭建本地Registry。

1\. 启动 Registry 容器，使用的镜像是`registry:2`

```
docker run -d -p 5000:5000 -v /myregistry:/var/lib/registry registry:2
```

解释下基本参数：

* -d 是后台启动容器。
* -p 将容器的 5000 端口映射到 Host 的 5000 端口。5000 是 registry 服务端口。端口映射后面讲。
* -v 将容器 /var/lib/registry 目录映射到 Host 的 /myregistry，用于存放镜像数据。-v 的使用后面讲。

2\. 通过 `docker tag` 重命名镜像，使之与 registry 匹配

```
docker tag yidao620/httpd:v1 localhost:5000/yidao620/httpd:v1
```

我们在镜像的前面加上了运行 registry 的主机名称和端口。

前面已经讨论了镜像名称由 repository 和 tag 两部分组成。
而 repository 的完整格式为：`[registry-host]:[port]/[username]/xxx`

只有 Docker Hub 上的镜像可以省略 `[registry-host]:[port]` 。

3\. 通过 `docker push` 上传镜像至本地 Registry

```
docker push localhost:5000/yidao620/httpd:v1
```

![](https://xnstatic-1253397658.file.myqcloud.com/docker19.png)

4\. 现在可以通过 `docker pull` 从本地Registry拉取镜像了

```
docker pull localhost:5000/yidao620/httpd:v1
```

除了镜像的名称长一些（包含 registry host 和 port），使用方式完全一样。

以上是搭建本地 registry 的简要步骤。当然 registry 也支持认证，https 安全传输等特性，
具体可以参考官方文档 <https://docs.docker.com/registry/configuration/>

## 删除和搜索镜像

删除镜像使用 `rmi ` 命令，只能删除 host 上的镜像，不会删除 registry 的镜像。

如果一个镜像对应了多个 tag，只有当最后一个 tag 被删除时，镜像才被真正删除。

```
docker rmi centos:latest
```

搜索镜像使用 `search` 命令，无需打开浏览器就能搜索 Docker Hub 中的镜像：

```
docker search httpd
```




