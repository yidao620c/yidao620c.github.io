---
title: Docker容器安装Redis集群
date: '2018-09-02 10:24:40 +0800'
toc: true
categories: [ 中间件 ]
tags: [ redis ]
---

Redis集群分两种模式，一种是Master-Slave模式，就是主从模式，一个master带多个slave，
另外一种是cluster模式，由多组master-slave组成。
<!-- more -->

## 主从模式

准备一个目录，比如`/root/redis-ms`

### 安装docker

这里省略

### 在docker库获取镜像：redis，ruby

```bash
docker pull redis
```

### 主redis服务

配置文件`redis_master.conf`

```
daemonize no
pidfile "/var/run/redis.pid"
port 6379
timeout 300
loglevel warning
logfile "redis.log"
databases 16
rdbcompression yes
dbfilename "redis.rdb"
dir "/data"
requirepass password
masterauth password
maxclients 10000
maxmemory 1000mb
maxmemory-policy allkeys-lru
appendonly no
appendfsync always
```

启动主redis服务

```bash
 docker run --name redis_master -v $(pwd)/redis_master.conf:/data/redis_master.conf --restart=always -d redis redis-server /data/redis_master.conf
```

### 从redis服务

配置文件`redis_slave.conf`

```
daemonize no
pidfile "/var/run/redis.pid"
port 6379
timeout 300
loglevel warning
logfile "redis.log"
databases 16
rdbcompression yes
dbfilename "redis.rdb"
dir "/data"
requirepass password
masterauth password
maxclients 10000
maxmemory 1000mb
maxmemory-policy allkeys-lru
appendonly no
appendfsync always
slaveof 172.17.0.4 6379
```

注意上面的IP地址172.17.0.4就是主redis服务容器地址，然后启动从redis服务：

```bash
docker run --name redis_slave -v $(pwd)/redis_slave.conf:/data/redis_slave.conf --restart=always -d redis:latest redis-server /data/redis_slave.conf
```

### 哨兵服务

配置文件sentinel.conf

```
daemonize no
port 26379
dir "/tmp"
sentinel monitor mymaster 172.17.0.4 6379 2
sentinel down-after-milliseconds mymaster 60000
sentinel auth-pass mymaster password
sentinel config-epoch mymaster 0
sentinel leader-epoch mymaster 0

# Generated by CONFIG REWRITE
sentinel known-slave mymaster 172.17.0.3 6379
sentinel current-epoch 0
```

启动哨兵服务：

```bash
docker run --name sentinel -v $(pwd)/sentinel.conf:/data/sentinel.conf --restart=always -d redis redis-sentinel /data/sentinel.conf
```

### 最后验证

```bash
docker inspect redis_master | grep IPA
docker run -it redis redis-cli -h 172.17.0.4
172.17.0.4:6379> info
NOAUTH Authentication required.
172.17.0.4:6379> auth password
OK
172.17.0.4:6379> info
# Server
redis_version:3.0.7
redis_git_sha1:00000000
redis_git_dirty:0
...
```

## 集群模式

### 准备环境

准备一个目录，比如`/root/redis-cluster`

### 安装docker

这里省略

### 在docker库获取镜像：redis，ruby

```bash
docker pull redis
docker pull ruby
```

### 下载最新redis源码包

```bash
wget https://github.com/antirez/redis/archive/unstable.zip
```

下载完成后解压到当前文件夹

### redis配置文件模板

找到一份原始的redis.conf文件，将其重命名为：redis-cluster.tmpl，并配置如下几个参数，此文件的目的是生成每一个redis实例的redis.conf:

```
# bind 127.0.0.1
port ${PORT}
daemonize no
dir /data/redis
appendonly yes
cluster-enabled yes
cluster-config-file nodes.conf
cluster-node-timeout 15000
```

## 构建一个redis集群管理镜像

```dockerfile
FROM ruby
MAINTAINER xn<xiongneng@gmail.com>
RUN gem install redis
RUN mkdir /redis
WORKDIR /redis
ADD ./redis-unstable /redis/redis-unstable
WORKDIR /redis/redis-unstable
RUN make && make install
RUN ls /redis/redis-unstable/src/redis-cli
RUN redis-cli --version
```

制作镜像： `docker build -t redis-cluster .`

## 创建集群

创建docker network

```bash
# 创建docker内部网络
docker network create redis-cluster-net
```

接下来编写自动脚本：

```bash
#!/bin/bash

# 创建 master 和 slave 文件夹
rm -rf ./master ./slave
for port in `seq 7000 7005`; do
    ms="master"
    if [ $port -ge 7003 ]; then
        ms="slave"
    fi
    mkdir -p ./$ms/$port/ && mkdir -p ./$ms/$port/data \
    && PORT=$port envsubst < ./redis-cluster.tmpl > ./$ms/$port/redis.conf;
done

# 运行docker redis 的 master 和 slave 实例
for port in `seq 7000 7005`; do
    ms="master"
    if [ $port -ge 7003 ]; then
        ms="slave"
    fi
    docker run -d -p $port:$port -p 1$port:1$port \
    -v $PWD/$ms/$port/redis.conf:/data/redis.conf \
    -v $PWD/$ms/$port/data:/data/redis \
    --restart always --name redis-$ms-$port --net redis-cluster-net \
    redis redis-server /data/redis.conf;
done

# 组装masters : slaves 节点参数
matches=""
for port in `seq 7000 7005`; do
    ms="master"
    if [ $port -ge 7003 ]; then
        ms="slave"
    fi
    matches=$matches$(docker inspect --format '{{(index .NetworkSettings.Networks "redis-cluster-net").IPAddress}}' "redis-$ms-${port}"):${port}" ";
done

# 创建docker-cluster
docker run -it --rm --net redis-cluster-net redis-cluster redis-cli --cluster create --cluster-replicas 1 $matches
```

运行后会让你确认一下，输入yes即可。

## 验证

进入容器：

```bash
docker exec -it redis-master-7000 /bin/bash
```

连接redis：

```bash
172.19.0.2:7000> set a aaa
-> Redirected to slot [15495] located at 172.19.0.4:7002
OK
```

注意上面的`Redirected to slot`，表明分配到集群中的其他机器上了。