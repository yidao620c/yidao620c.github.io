---
title: Docker每天学一点04 - Dockerfile
date: 2019-03-05 15:17:09
toc: true
categories: [ k8s ]
tags: [ docker ]
---

是时候系统学习 Dockerfile 了，下面介绍 Dockerfile 中最常用的指令，完整列表和说明可参看官方文档。
<!-- more -->

## FROM

指定 base 镜像。

## MAINTAINER

设置镜像的作者，可以是任意字符串。

## COPY

将文件从 build context 复制到镜像。

COPY 支持两种形式：

1. COPY src dest
2. COPY ["src", "dest"]

注意：src 只能指定 build context 中的文件或目录。

如果dest在容器中不存在，会被自动创建。经过我的实际测试，存在以下情况：

1. src是文件，而dest以/结尾，那么创建dest/文件夹，并将src复制到文件夹中
1. src是文件，而dest不以/结尾，那么创建dest父目录，并将src复制到父目录并改名为dest
1. src是文件夹，不管dest是否/结尾，都会创建dest文件夹，并将src目录下的所有内容复制到dest文件夹中（不包括src目录本身）。

对文件而言，dest是否以/结尾非常重要。

## ADD

与 COPY 类似，从 build context 复制文件到镜像。
不同的是，如果 src 是归档文件（tar, zip, tgz, xz 等），文件会被自动解压到 dest。

经测试发现，如果是归档文件，不管dest是否以/结尾，最终都会创建dest/并将归档文件复制到这个文件夹后解压出来。

## ENV

设置环境变量，环境变量可被后面的指令使用。例如：

```
ENV MY_VERSION 1.3
RUN apt-get install -y mypackage=$MY_VERSION
```

## EXPOSE

指定容器中的进程会监听某个端口，Docker 可以将该端口暴露出来。我们会在容器网络部分详细讨论。

## VOLUME

将文件或目录声明为 volume。

## WORKDIR

为后面的 RUN, CMD, ENTRYPOINT, ADD 或 COPY 指令设置镜像中的当前工作目录。

## RUN

在容器中运行指定的命令。

## CMD

容器启动时运行指定的命令。

Dockerfile 中可以有多个 CMD 指令，但只有最后一个生效。CMD 可以被 docker run 之后的参数替换。

## ENTRYPOINT

设置容器启动时运行的命令。

Dockerfile 中可以有多个 ENTRYPOINT 指令，但只有最后一个生效。
CMD 或 docker run 之后的参数会被当做参数传递给 ENTRYPOINT。

下面我们来看一个较为全面的 Dockerfile：

```
# my centos dockerfile
FROM centos
MAINTAINER xn@enzhico.com
WORKDIR /work
RUN touch README.md
COPY ["README.md", "."]
ADD ["test.tar.gz", "."]
ENV WELCOME "welcome you, Jim"
```

然后执行步骤：

1. 构建前确保 build context 中存在需要的文件。
2. 依次执行 Dockerfile 指令，完成构建。

![](https://xnstatic-1253397658.file.myqcloud.com/docker15.png)

运行容器，验证镜像内容：

![](https://xnstatic-1253397658.file.myqcloud.com/docker16.png)

可以看到我的工作目录是/work，并且里面的内容也都正确复制和解压缩了。

进入容器，当前目录即为 WORKDIR。如果 WORKDIR 不存在，Docker 会自动为我们创建。

ENV 指令定义的环境变量在容器里面已经生效。

## RUN、CMD 和 ENTRYPOINT

RUN、CMD 和 ENTRYPOINT 这三个 Dockerfile 指令看上去很类似，很容易混淆。

简单的说：

1. RUN 执行命令并创建新的镜像层，RUN 经常用于安装软件包。
2. CMD 设置容器启动后默认执行的命令及其参数，但 CMD 能够被 docker run 后面跟的命令行参数替换。
3. ENTRYPOINT 配置容器启动时运行的命令。

Shell 和 Exec 格式

我们可用两种方式指定 RUN、CMD 和 ENTRYPOINT 要运行的命令：Shell 格式和 Exec 格式，
二者在使用上有细微的区别。

Shell 格式:

```
RUN apt-get install python3  
CMD echo "Hello world"  
ENTRYPOINT echo "Hello world" 
```

当指令执行时，shell 格式底层会调用 `/bin/sh -c <command>`

Exec 格式：

```
RUN ["apt-get", "install", "python3"]  
CMD ["/bin/echo", "Hello world"]  
ENTRYPOINT ["/bin/echo", "Hello world"]
```

当指令执行时，会直接调用 <command>，不会被 shell 解析。

如果希望使用环境变量，照如下修改:

```
ENV name Cloud Man  
ENTRYPOINT ["/bin/sh", "-c", "echo Hello, $name"]
```

最佳实践

1. 使用 RUN 指令安装应用和软件包，构建镜像。
2. 如果 Docker 镜像的用途是运行应用程序或服务，比如运行一个 MySQL，
   应该优先使用 Exec 格式的 ENTRYPOINT 指令。CMD 可为 ENTRYPOINT 提供额外的默认参数，
   同时可利用 docker run 命令行替换默认参数。
3. 如果想为容器设置默认的启动命令，可使用 CMD 指令。用户可在 docker run 命令行中替换此默认命令。
4. apt-get update 和 apt-get install 被放在一个 RUN 指令中执行，这样能够保证每次安装的是最新的包

在书籍《第一本Docker书》上面举了一个例子，组合使用ENTRYPOINT和CMD指令来完成一些巧妙的工作。
比如，我们可以在Dockerfile指定如下内容：

```
ENTRYPOINT ["/usr/bin/nginx"]
CMD ["-h"]
```

当我们启动容器时候，任何命令行中指定参数会被传递给nginx守护进程，比如我们指定 `-g "daemon off"`让nginx以前台方式运行。
如果用户不指定任何启动命令参数，那么CMD指令的-h会被传递给nginx守护进程，显示帮助信息。这样就能利用CMD实现有个默认的启动参数。
