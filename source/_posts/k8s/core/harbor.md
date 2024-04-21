---
title: 安装Docker私有仓库Harbor
date: '2020-12-31 12:21:15 +0800'
toc: true
categories: [ k8s ]
tags: [ k8s, harbor ]
---

Harbor是VMware公司开源的企业级公司开源的企业级DockerRegistry项目， 项目地址为<https://github.com/vmware/harbor>。
其目标是帮助用户迅速搭建一个企业级的Docker镜像服务。它以Docker公司开源的registry为基础，提供了管理UI， 基于角色的访问控制(Role Based Access Control)
，LDAP集成、审计日志、管理界面、自我注册、 镜像复制等企业用户需求的功能，同时还原生支持中文。Harbor的每个组件都是以Docker容器的形式构建， 使用Docker
Compose来对它进行部署。用于部署Harbor的Compose模板位于`/Deployer/docker-compose.yml`， 由5个容器组成，这几个容器通过Docker link的形式连接在一起，在容器之间通过容器名字互相访问。
对终端用户而言，只需要暴露 Proxy（即Nginx）的服务端口。

* Proxy：由Nginx 服务器构成的反向代理。
* Registry：由Docker官方的开源 registry镜像构成的容器实例。
* UI：即架构中的 core services，构成此容器的代码是 Harbor 项目的主体。
* MySQL：由官方 MySQL 镜像构成的数据库容器。
* Log：运行着 rsyslogd 的容器，通过 log-driver 的形式收集其他容器的日志。

Harbor官方地址：https://goharbor.io/

<!--more-->

## 下载Harbor

前提是已安装Python、Docker和Docker Compose。安装官方最新版本即可。

从官网上面下载最新的离线安装包，目前最新的是`harbor-offline-installer-v2.1.3.tgz`。从github上面下载下来。 如果还想校验文件完整性，可以下载`*.asc`文件。然后执行如下校验命令

```bash
gpg --keyserver hkps://keyserver.ubuntu.com --recv-keys 644FF454C0B4115C
gpg -v --verify harbor-offline-installer-v2.1.3.tgz.asc
```

## 配置证书

生产环境最好使用HTTPS协议访问Harbor，因此需要有证书。这里说明通过openssl命令生成自签名证书。

```bash
# 定义变量
domain="xnharbor.com"
# 创建证书目录
mkdir -p /data/cert/ && cd /data/cert/
# 1、生成CA私钥
openssl genrsa -out ca.key 4096
# 2、生成CA根证书
openssl req -x509 -new -nodes -sha512 -days 3650 \
    -subj "/C=CN/ST=Beijing/L=Beijing/O=example/OU=Personal/CN=${domain}" \
    -key ca.key -out ca.crt
# 3、生成服务私钥
openssl genrsa -out ${domain}.key 4096
# 4、生成证书请求文件
openssl req -sha512 -new \
    -subj "/C=CN/ST=Beijing/L=Beijing/O=example/OU=Personal/CN=${domain}" \
    -key ${domain}.key -out ${domain}.csr
# 5、生成X509 V3证书的自定义配置文件，指定subjectAltName
cat > v3.ext <<EOF
authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
extendedKeyUsage = serverAuth
subjectAltName = @alt_names

[alt_names]
DNS.1=${domain}
DNS.2=xnharbor
DNS.3=harbor
EOF
# 6、签发自签名服务证书
openssl x509 -req -sha512 -days 3650 \
    -extfile v3.ext \
    -CA ca.crt -CAkey ca.key -CAcreateserial \
    -in ${domain}.csr -out ${domain}.cert
# 7、将CA根证书、服务私钥和证书文件复制到docker的证书目录中
mkdir /etc/docker/certs.d/${domain}/
/bin/cp ${domain}.cert /etc/docker/certs.d/${domain}/
/bin/cp ${domain}.key /etc/docker/certs.d/${domain}/
/bin/cp ca.crt /etc/docker/certs.d/${domain}/
# 8、重启docker引擎
systemctl restart docker
```

## 修改配置文件harbor.yml

解压后的安装目录有个配置模板文件`harbor.yml.tmpl`，将其复制一份为`harbor.yml`，下面说明一下必须参数。

> hostname: 主机名或IP地址。这是访问Harbor的地址
> https.port: HTTPS端口号，默认为443
> https.certificate: HTTPS证书路径
> https.private_key: HTTPS证书私钥路径
> internal_tls.enabled: 是否在内部组件之间通信启用TLS安全通道
> internal_tls.dir: 内部证书和私钥的所在目录
> harbor_admin_passwor: 初始管理员密码。默认管理员admin/Harbor12345
> database.password: 本地PostgreSQL数据库的root密码，最好修改这个值
> database.max_idle_conns: 连接池中最大空闲连接数
> database.max_open_conns: 连接池中最大连接数
> data_volume: 数据存储卷。默认为/data/

## 执行安装

配置好了之后，直接执行`./install.sh`即可。看到下面的输出就是成功了

```
Successfully called func: create_root_cert
Generated configuration file: /compose_location/docker-compose.yml
Clean up the input dir

[Step 5]: starting Harbor ...
Building with native build. Learn about native build in Compose here: https://docs.docker.com/go/compose-native-build/
Creating network "harbor_harbor" with the default driver
Creating harbor-log ... done
Creating registry      ... done
Creating harbor-portal ... done
Creating registryctl   ... done
Creating redis         ... done
Creating harbor-db     ... done
Creating harbor-core   ... done
Creating nginx             ... done
Creating harbor-jobservice ... done
✔ ----Harbor has been installed and started successfully.----
```

## 验证

本地配置好hosts域名映射后，通过浏览器访问域名 <https://xnharbor.com/> 即可。输入管理员账号口令登录。
![img.png](https://xnstatic-1253397658.file.myqcloud.com/20201231-01.png)

还需要验证一下上传下载镜像的基本功能。首先创建一个test的项目。

首先指定镜像仓库地址

```json
{
    "registry-mirrors": ["https://30y4mr57.mirror.aliyuncs.com", "https://xnharbor.com"]
}
```

拉取一个hello-world镜像，再推到私有仓库中。

```bash
docker pull hello-world
docker tag hello-world xnharbor.com/test/hello-world:0.0.1
docker login xnharbor.com
docker push xnharbor.com/test/hello-world:0.0.1
# 先把已有的本地镜像删了
docker rmi xnharbor.com/test/hello-world:0.0.1 hello-world
# 最后再次下载看看
docker pull xnharbor.com/test/hello-world:0.0.1
```

