---
title: CentOS7搭建ngrok服务器
date: 2017-12-29 09:48:19 +0800
toc: true
categories: [ linux ]
tags: [ linux ]
---

ngrok是一个反向代理，它能够让你本地的web服务或tcp服务通过公共的端口和外部建立一个安全的通道，
使得外网可以访问本地的计算机服务。也就是说，我们提供的服务（比如web站点）无需搭建在外部服务器，
只要通过ngrok把站点映射出去，别人即可直接访问到我们的服务。

有做过微信公众号开发的人，对它应该不陌生。因为用户跟微信公众号产生的交互行为，微信会把用户的相关信息推送到我们自己的服务器，
如果服务在本地，那微信当然无法推送给我们，这使得开发功能的时候调试相当麻烦。我们可以使用ngrok把本地站点映射出去，解决这个问题。

另外如果我们想把本地开发时候的系统临时给外网用户看，无需部署到服务器上面去就可以，非常方便。

ngrok是开源的，官网地址：<https://github.com/inconshreveable/ngrok>

下面，我们开始搭建ngrok服务。操作系统为CentOS 7.2
<!-- more -->

## 准备工作

搭建ngrok服务需要有一个外网服务器及一个域名解析到外网服务器上，我已经有了一个`xncoding.com`域名，并且拥有一台腾讯云主机。

在腾讯云主机的域名解析处，配置2个A记录，比如我新建2个`ngrok.xncoding.com` 和 `*.ngrok.xncoding.com` 解析到vps服务器上。

![](https://xnstatic-1253397658.file.myqcloud.com/ngrok01.png)

## 搭建ngrok服务

### 安装go语言环境

ngrok是基于go语言开发的，所以需要先安装go语言开发环境，CentOS可以使用yum安装：

```bash
yum install golang
```

### 安装git

默认的git版本太低了，需要升级到git2.5，具体步骤如下：

```bash
sudo yum remove git
sudo yum install epel-release
sudo yum install https://centos7.iuscommunity.org/ius-release.rpm
sudo yum install git2u
```

`git --version`，返回 `git version 2.5.0`，安装成功。

### 下载ngrok源码

新建一个目录，并clone一份源码：

```bash
mkdir ~/go/src/github.com/inconshreveable
cd ~/go/src/github.com/inconshreveable
git clone https://github.com/inconshreveable/ngrok.git
export GOPATH=~/go/src/github.com/inconshreveable/ngrok
```

### 生成自签名证书

使用ngrok.com官方服务时，我们使用的是官方的SSL证书。自己建立ngrok服务，需要我们生成自己的证书，并提供携带该证书的ngrok客户端。

证书生成过程需要有自己的一个基础域名，比如我的就是`ngrok.xncoding.com`。

```bash
$ cd ngrok
$ openssl genrsa -out rootCA.key 2048
$ openssl req -x509 -new -nodes -key rootCA.key -subj "/CN=ngrok.xncoding.com" -days 5000 -out rootCA.pem
$ openssl genrsa -out device.key 2048
$ openssl req -new -key device.key -subj "/CN=ngrok.xncoding.com" -out device.csr
$ openssl x509 -req -in device.csr -CA rootCA.pem -CAkey rootCA.key -CAcreateserial -out device.crt -days 5000
```

执行完成以上命令后，在ngrok目录下，会新生成6个文件：

```
-rw-r--r-- 1 root root   1001 Dec 29 11:53 device.crt
-rw-r--r-- 1 root root    903 Dec 29 11:44 device.csr
-rw-r--r-- 1 root root   1675 Dec 29 11:44 device.key
-rw-r--r-- 1 root root   1679 Dec 29 11:44 rootCA.key
-rw-r--r-- 1 root root   1119 Dec 29 11:44 rootCA.pem
-rw-r--r-- 1 root root     17 Dec 29 11:53 rootCA.srl
```

我们在编译可执行文件之前，需要把生成的证书分别替换到 `assets/client/tls`和`assets/server/tls`中，
这两个目录分别存放着ngrok和ngrokd的默认证书。

```bash
$ cp rootCA.pem assets/client/tls/ngrokroot.crt
$ cp device.crt assets/server/tls/snakeoil.crt
$ cp device.key assets/server/tls/snakeoil.key
```

### 使用lets encrypt免费证书

如果想让浏览器不弹出提示，最好不要使用自签名证书，现在lets encrypt推出泛域名证书了，所以可以先申请个免费域名证书。

客户端用证书 ：

```bash
cd ngrok
cp /etc/letsencrypt/live/xncoding.com/chain.pem assets/client/tls/ngrokroot.crt
```

服务器端用证书：

```bash
cp /etc/letsencrypt/live/xncoding.com/cert.pem assets/server/tls/snakeoil.crt
cp /etc/letsencrypt/live/xncoding.com/privkey.pem assets/server/tls/snakeoil.key
```

### 编译ngrokd和ngrok

首先需要知道，ngrokd 为服务端的执行文件，ngrok为客户端的执行文件。

接下来我们来编译ngrokd，在ngrok目录下，执行如下命令：

```bash
$ make release-server
```

编译过程需要等待一会，因为需要通过git安装相关依赖包。如果提示没有权限，使用 sudo 命令来安装。

由于客户端的平台版本较多，我们需要交叉编译来选择生成的平台。
以windows、arm、linux版本编译，如下：

```
$ GOOS=linux GOARCH=amd64 make release-client
$ GOOS=windows GOARCH=amd64 make release-client
$ GOOS=linux GOARCH=arm make release-client
```

不同平台使用不同的 GOOS 和 GOARCH，GOOS为go编译出来的操作系统 (windows,linux,darwin)，GOARCH, 对应的构架 (386,amd64,arm)

通过上面的步骤，将生成所有客户端文件，客户端文件放在对于的文件夹中，如windows 64位的为：windows_amd64，linux客户端在bin目录下的ngrok文件。

完成之后，把相应的客户端文件使用SFTP或其他方式分发到客户端电脑上面，比如我用的windows电脑，就把`windows_amd64/ngrok.exe`
文件复制过去。

### 启动ngrokd服务器

请将 bin/ngrokd 放入PATH环境变量中，启动命令：

```bash
nohup ngrokd -domain=ngrok.xncoding.com -httpAddr=:5442 -httpsAddr=:5443 -tunnelAddr=":4443" &
```

`-domain`为你的服务域名，`-httpAddr`为http服务端口地址，访问形式为`xxx.ngrok.xncoding.com:5442`
，也可设置为80默认端口，`-httpsAddr`为https服务，同上。

ngrokd还会开一个端口用来跟客户端通讯（可通过`-tunnelAddr=":xxx"` 指定），如果你配置了 iptables 规则，需要放行这个通讯端口(
4443)上的 TCP 协议。

```bash
firewall-cmd --zone=public --add-port=4443/tcp --permanent
firewall-cmd --reload
```

### Nginx配置80端口转发

我们在微信开发时候不允许使用端口访问，那么最好使用nginx反向代理转发，首先申请一个`demo.ngrok.xncoding.com`
的免费证书，然后修改nginx配置如下：

```nginx
server {
    listen       80;
    server_name  demo.ngrok.xncoding.com;
    return       301 https://demo.ngrok.xncoding.com$request_uri;
}

server {
    listen       443 ssl http2;
    server_name  demo.ngrok.xncoding.com;

    charset utf-8;

    ssl_certificate /etc/letsencrypt/live/demo.ngrok.xncoding.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/demo.ngrok.xncoding.com/privkey.pem;
    ssl_trusted_certificate /etc/letsencrypt/live/demo.ngrok.xncoding.com/chain.pem;

    access_log /var/log/nginx/ngrok.log main;
    error_log /var/log/nginx/ngrok_error.log error;

    location / {
        proxy_pass http://127.0.0.1:5442;
        proxy_redirect off;
        proxy_set_header Host       $http_host:5442;
        proxy_set_header X-Real-IP  $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

但是！这里就有一个很烦躁的地方了，ngrokd 里面有一层自己的 Host 处理，于是 `proxy_set_header Host` 必须带上 ngrokd 所监听的端口，
否则就算请求被转发到对应端口上， ngrokd 也不会正确的处理。

### 启用客户端

在刚刚复制过来的ngrok.exe客户端文件夹中，新建一个客户端配置`ngrok.cfg`：

```
server_addr: "ngrok.xncoding.com:4443"
trust_host_root_certs: false
```

本地启动一个SpringBoot的WEB工程，端口8092，然后通过下面命令启动客户端：

```
ngrok.exe -subdomain demo -config=ngrok.cfg -log=log.txt 8092
```

看到下面的画面说明连接成功了：

![](https://xnstatic-1253397658.file.myqcloud.com/ngrok03.png)

访问页面，浏览器中输入：`https://demo.ngrok.xncoding.com`，成功访问本地SpringBoot站点内容。

浏览器输入：`127.0.0.1:4040` 查看页面请求情况：

![](https://xnstatic-1253397658.file.myqcloud.com/ngrok02.png)

## 烦恼的事情

带上端口号又会导致了另一个操蛋的问题：你请求的时候是`demo.ngrok.xncoding.com`，
你在 web 应用中获取到的 Host 是 `demo.ngrok.xncoding.com:5442`，
如果你的程序里面有基于 Request Host 的重定向，就会被重定向到 `demo.ngrok.xncoding.com:5442` 下面去。

要完美的解决这个端口的问题，就需要让 ngrokd 直接监听 80 端口，或者使用Docker容器的端口映射来解决。

## 使用Docker

上面我讲到自己手动搭建的时候出现的端口问题，没办法解决。
一般80端口早就被占用了，不可能就给你ngrok使用，最完美的方式是使用Docker + Nginx的方式。

### 安装docker

参考我的这篇[Docker入门](https://www.xncoding.com/2017/04/01/docker/docker01.html)来安装docker。

### 构建镜象

这里使用的是[hteen/docker-ngrok](https://github.com/hteen/docker-ngrok)

```bash
git clone https://github.com/hteen/docker-ngrok.git
cd docker-ngrok
docker build -t hteen/ngrok .
```

这里需要等待一段时间下载

### Docker容器的https

关于 https 的支持

由于 ngrok 工作是通过分配 subdomain 的方式，所以我们实际使用到的域名都是 `ngrok.xncoding.com` 的子域名，
如 `demo.ngrok.xncoding.com` 如果要对这个子域名启用 https 服务，那么至少需要三点支持：

1. ngrok 支持 https， 这个默认就是开启的
1. `demo.ngrok.xncoding.com` 也需要有证书或包含在一个泛域名证书中
1. 浏览器（或其他终端）信任 `demo.ngrok.xncoding.com` 的根证书

好消息是现在lets encrypt支持通配符域名了，所以很简单，具体怎么申请，请参考我博客中的nginx相关文章。

这里请先申请`ngrok.xncoding.com`的通配符证书。

申请好之后，增加配置`/etc/nginx/conf.d/ngrok.conf` 添加反向代理配置：

```
map $scheme $proxy_port {
    "http"  "5442";
    "https" "5443";
    default "5442";
}

server {
    listen      80;
    listen      443;
    server_name ngrok.xncoding.com *.ngrok.xncoding.com;

    location / {
        proxy_pass  $scheme://127.0.0.1:$proxy_port;
    }

    ssl on;
    ssl_certificate /etc/letsencrypt/live/ngrok.xncoding.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/ngrok.xncoding.com/privkey.pem;

    proxy_set_header    X-Real-IP $remote_addr;
    proxy_set_header    Host $http_host;
    proxy_set_header    X-Forwarded-For $proxy_add_x_forwarded_for;

    access_log off;
    log_not_found off;
}
```

### 运行镜象

上一步已经申请`ngrok.xncoding.com`这个域名的通配符lets encrypt证书，然后修改脚本`server.sh`

```
-tlsKey=/etc/letsencrypt/live/ngrok.xncoding.com/privkey.pem -tlsCrt=/etc/letsencrypt/live/ngrok.xncoding.com/fullchain.pem
```

也就是将之前的证书变量改成你实际的证书路径即可。

然后运行：

```bash
docker run -idt --name ngrok-server \
-p 5442:80 -p 5443:443 -p 4443:4443 \
-v /data/ngrok:/myfiles \
-e DOMAIN='ngrok.xncoding.com' -e HTTP_ADDR=':80' -e HTTPS_ADDR=':443' hteen/ngrok /bin/sh /server.sh
```

如果在腾讯云主机上面，还需要本级防火墙放行4443端口，以及在腾讯云安全组中也要放开4443端口。

```bash
1.查看已开放的端口(默认不开放任何端口)
firewall-cmd --list-ports
2.开启4443端口
firewall-cmd --zone=public --add-port=4443/tcp --permanent
3.重启防火墙
systemctl restart firewalld
```

这里会把主机的5442端口映射到Docker容器中的80端口，讲5443端口映射到443端口，同时将本机的/data/ngrok文件夹映射到docker容器的/myfiles目录。

运行后，会要等一段时间，因为要编译客户端。一直等到/data/ngrok/目录里面有/bin目录就OK了。

### 注意事项

因为ngrok有心跳机制，每次心跳均会产生日志，所以以docker方式运行，会产生很多日志。实测试中，大概每个星期会产生100M的日志文件。

查年docker日志文件位置`sudo docker inspect <id> | grep LogPath`

查看大小`sudo ls -lh /var/lib/docker/containers/<id>/<id>-json.log`

### 运行客户端

在`/data/ngrok/bin/`目录下会生成客户端程序，每个平台的版本都有。以windows64位来说，
在windows_amd64目录下，拷贝到自己的windows电脑上。

新建配置文件ngrok.cfg，跟ngrok.exe同级目录，里面的内容跟之前讲的一样：

```
server_addr: "ngrok.xncoding.com:4443"
trust_host_root_certs: false
```

然后打开windows的命令行，cd到`ngrok.exe`所在的目录中，到这个运行：

```
ngrok -config=ngrok.cfg -subdomain=demo -log=log.txt 8092
```

或者为了方便，在`ngrok.exe`所在的目录中新建一个`run.bat`文件，内容如下：

```
@echo off
ngrok.exe -config=ngrok.cfg -subdomain=demo -log=log.txt 8092
```

上面的subdomain是你想去访问域名前缀，后面的端口是你本机应用启动端口。

看到下面的结果表示成功了：

![](https://xnstatic-1253397658.file.myqcloud.com/ngrok04.png)

然后再打开`http://demo.ngrok.xncoding.com`看看，发现不会像之前那样出现端口了。

## 国内免费的ngrok

如果你自己没VPS，或者你机子上面80端口已经被nginx占用不想搞了，就直接使用免费的ngrok吧，
我推荐你使用<https://www.ngrok.cc/>。

比如我自己弄了个`yidao620.free.ngrok.cc`，启动本地客户端后，映射到本地的8092端口了，也还不错。

## 参考文章

* [从零教你搭建ngrok服务器](https://morongs.github.io/2016/12/28/dajian-ngrok/)
* [ngrok使用自己的证书通过https访问](https://www.jianshu.com/p/4b03fb532145)
* [搭建并配置优雅的 ngrok 服务实现内网穿透](https://yii.im/posts/pretty-self-hosted-ngrokd/)
* [使用Docker搭建Ngrok服务器实现内网穿透](https://blog.fengcl.com/2017/05/24/how-to-use-docker-build-ngrok-to-network-penetrate/)
* [搭建自己的Ngrok服务器, 并与Nginx并存](https://fengqi.me/unix/409.html)
