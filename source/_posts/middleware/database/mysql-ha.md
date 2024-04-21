---
title: centos7配置mysql主从复制
date: 2016-09-16 22:22:22 +0800
toc: true
categories: [中间件]
tags: [ mysql ]
---

常见的高可用MySQL解决方案有主从复制，主主复制，Heartbeat/SAN高可用，MySQL Cluster高可用等等。
这里我使用最简单的主从复制高可用方案，也是Mysql内置的功能。

mysql主从复制原理：

1. master将数据改变记录到二进制日志(binary log)中,也即是配置文件log-bin指定的文件(这些记录叫做二进制日志事件)
2. slave将master的binary log events拷贝到它的中继日志(relay log)
3. slave重做中继日志中的事件,将改变反映它自己的数据(数据重演)

其实就是根据主数据库的日志文件来进行同步。
<!-- more -->

## 支持类型

1. 基于语句的复制:在主服务器上执行的SQL语句,在从服务器上执行同样的语句.MySQL默认采用基于语句的复制,效率比较高
2. 基于行的复制:把改变的内容直接复制过去,而不关心到底改变该内容是由哪条语句引发的 . 从mysql5.0开始支持
3. 混合类型的复制: 默认采用基于语句的复制,一旦发现基于语句的无法精确的复制时,就会采用基于行的复制.

## 前置条件

1. 主DB server和从DB server数据库的版本一致
2. 主DB server和从DB server数据库数据一致[可以把主的备份在从上还原]
3. 主DB server开启二进制日志,主DB server和从DB server的server_id都必须唯一

## 详细步骤

主服务器node200：192.168.212.200

从服务器node201：192.168.212.201（可以设置多个从服务器）

1\. 两台机器的的selinux都是disable

2\. 修改主DB server的配置文件(/etc/my.cnf)，开启日志功能,设置server_id值,保证唯一[node200为主DB server]

```bash
[root@node200 ~]# vi /etc/my.cnf
# 修改配置文件里,下面两个参数：
# 设置server_id,一般建议设置为IP,或者再加一些数字
server_id = 200
# 开启二进制日志功能,可以随便取,最好有含义
log-bin=mysql3306-bin
```

3\. 启动数据库服务器,并登陆数据库,授予相应的用户用于同步

```bash
mysql -uroot -p
mysql> GRANT REPLICATION SLAVE,RELOAD,SUPER ON *.* TO 'slave1'@'192.168.212.201' IDENTIFIED BY 'winstore';
# 刷新授权表信息
mysql> FLUSH PRIVILEGES;
# 查看记下position 号和日志文件名(很重要)
mysql> show master status;
+----------------------+----------+--------------+------------------+-------------------+
| File                 | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+----------------------+----------+--------------+------------------+-------------------+
| mysql3306-bin.000002 |      399 |              |                  |                   |
+----------------------+----------+--------------+------------------+-------------------+
1 row in set (0.00 sec)
```

4\. 为保证主DB server和从DB server的数据一致,这里采用主备份,从还原来实现初始数据一致

```bash
# 主DB上面登录
# 临时锁表
mysql> flush tables with read lock;
# 我这里实行的全库备份,在实际中,我们可能只同步某一个库,可以只备份一个库
# 新开一个终端,执行如下操作
[root@node200 ~]# mysqldump -uroot -p --all-databases > /tmp/mysql.sql
# 解锁
mysql> unlock tables;
# 将备份的数据传送到从机上,用于恢复
[root@node200 ~]# scp /tmp/mysql.sql root@192.168.212.201:/tmp
```

5\. 从数据库配置文件只需修改一项,其余用命令行做

```bash
[root@node201 ~]# vi /etc/my.cnf
# 设置server_id,一般建议设置为IP
server_id = 201
```

6\.启动从数据库,还原备份数据

```bash
# 启动数据库
[root@node201 ~]# systemctl restart mysqld.service
# 还原主DB server备份的数据
[root@node201 ~]# mysql -uroot -p < /tmp/mysql.sql
```

7\. 登陆从数据库,添加相关参数(主DBserver的ip/端口/同步用户/密码/position号/读取哪个日志文件)

```bash
[root@node201 ~]# mysql -uroot -p
mysql> change master to
    -> master_host='192.168.212.200',
    -> master_user='slave1',
    -> master_password='winstore',
    -> master_port=3306,
    -> master_log_file='mysql3306-bin.000002',
    -> master_log_pos=399;
# 开启主从同步
mysql> start slave;
# 查看主从同步状态
mysql> show slave status\G;
# 主要看以下两个参数：[这两个参数如果是yes就表示主从同步正常]
Slave_IO_Running: Yes
Slave_SQL_Running: Yes
```

8\. 下面大家就可以在主DB server上新建一个表,看是否能同步到从DB server上,我这里就不测试了

查看server_id：

```bash
mysql> show variables like '%server_id%';
```
