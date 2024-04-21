---
title: Docker安装常用软件
date: 2019-03-30 12:35:12
toc: true
categories: [ k8s ]
tags: [ docker ]
---

这里总结常用软件的容器化安装步骤，环境为CentOS7。

环境准备要先安装Docker软件，配置好国内加速镜像，这个可以参考我的Docker教程入门篇。这里不再多讲。

这里演示如何在CentOS7上面通过Docker安装MySQL8版本。

拉取镜像文件：

```
docker pull mysql/mysql-server
```

启动镜像文件：

```
docker run -d -p 13306:3306 --name mysql \
-v /data/mysql/conf.d:/etc/mysql/conf.d \
-v /data/mysql/data:/var/lib/mysql \
mysql/mysql-server
```

<!-- more -->

首先执行下面命令查看容器日志，找到MySql的root账户的密码：

```
docker logs mysql

[Entrypoint] GENERATED ROOT PASSWORD: ORAk@KwYth0LQuktYjmUw!UkxEs
```

看到类似上面那一行，复制一下密码。执行下面命令进入到容器中并连接MySQL：

```
docker exec -it mysql bash
mysql -uroot -p
```

然后执行下面语句修改root密码：

```
alter user 'root'@'localhost' identified by 'vs_654321';
```

修改完root密码后，可以使用下面代码切换到mysql数据库并查看用户信息：

```
use mysql
select user,host from user;

mysql> select user,host from user;
+------------------+-----------+
| user             | host      |
+------------------+-----------+
| healthchecker    | localhost |
| mysql.infoschema | localhost |
| mysql.session    | localhost |
| mysql.sys        | localhost |
| root             | localhost |
+------------------+-----------+

```

可以看到root的host为localhost，说明root账户不能被外部连接，现在来创建一个新的用户，
并赋相关的权限让外部可以连接，一次执行下面语句：

```
CREATE USER 'test'@'localhost' IDENTIFIED BY 'test123456';
GRANT ALL PRIVILEGES ON *.* TO 'test'@'localhost' WITH GRANT OPTION;
CREATE USER 'test'@'%' IDENTIFIED BY 'test123456';
GRANT ALL PRIVILEGES ON *.* TO 'test'@'%' WITH GRANT OPTION;
ALTER USER 'test'@'%' IDENTIFIED WITH mysql_native_password BY 'test123456';
```

然后用test用户去连接数据库即可。
