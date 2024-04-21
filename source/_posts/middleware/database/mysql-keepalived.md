---
title: 基于keepalived实现mysql双主
date: 2016-09-19 21:17:24 +0800
toc: true
categories: [中间件]
tags: [ mysql ]
---

MySQL的高可用方案一般有如下几种：

mysql自带的主从、keepalived+双主、MHA、MMM、Heartbeat+DRBD、PXC、Galera Cluster

我在前面一篇博客讲解过mysql自带的主从复制，不过更加常用的是keepalived+双主，MHA和PXC。
对于小公司，一般推荐使用keepalived+双主，简单。

当主从复制正在进行中时，如果想查看从库两个线程运行状态，
可以通过执行在从库里执行 `show slave status\G` 语句，以下的字段可以给你想要的信息：

```
Master_Log_File       — 上一个从主库拷贝过来的binlog文件
Read_Master_Log_Pos   — 主库的binlog文件被拷贝到从库的relay log中的位置
Relay_Master_Log_File — SQL线程当前处理中的relay log文件
Exec_Master_Log_Pos   — 当前binlog文件正在被执行的语句的位置
```

<!-- more -->

整个主从复制的流程可以通过以下图示理解：

![](https://xnstatic-1253397658.file.myqcloud.com/mysql-copy.png)

步骤说明：

1. 步骤一：主库db的更新事件(update、insert、delete)被写到binlog
1. 步骤二：从库发起连接，连接到主库
1. 步骤三：此时主库创建一个binlog dump thread，把binlog的内容发送到从库
1. 步骤四：从库启动之后，创建一个I/O线程，读取主库传过来的binlog内容并写入到relay log
1. 步骤五：还会创建一个SQL线程，从relay log里面读取内容，从Exec_Master_Log_Pos位置开始执行读取到的更新事件，将更新内容写入到slave的db

注：上面的解释是解释每一步做了什么，整个mysql主从复制是异步的，不是按照上面的步骤执行的。

## 双主高可用原理

在主主复制结构中，两台的任何一台上面的数据库存发生了改变都会同步到另一台服务器上，
这个改变是基于sql语句的改变，如果删除系统数据库源文件或删除后新创建同名MYSQL表实现同步则无效。
这样两台服务器互为主从，并且都能向外提供服务，这就比使用主从复制具有更好的性能。

虽然理论上，双主只要数据不冲突就可以工作的很好，但实际情况中还是很容发生数据冲突的，比如在同步完成之前，
双方都修改同一条记录。因此在实际中，最好不要让两边同时修改，即逻辑上仍按照主从的方式工作。

所以可以借助keepalived这样的软件实现自动主从切换，工作时候仍然按照主从方式复制，
并且只允许一台主节点写数据，这样就不会产生冲突了，一旦主库出现问题可以自动切换。

## 部署环境

 HOSTNAME | ROLE    | IP         | OS      | software            
----------|---------|------------|---------|---------------------
 node001  | master1 | 172.17.0.2 | centos7 | mysql5.7/keepalived 
 node002  | master2 | 172.17.0.3 | centos7 | mysql5.7/keepalived 
 node003  | client  | 172.17.0.1 | centos7 | mysql-client5.7     

VIP：172.17.0.4

由于公司做开发只有一台腾讯云测试机器，所以为了试验我使用了Docker容器来部署集群。

node003是腾讯云主机，作为mysql的客户端来测试集群，docker网络ip为172.17.0.1。

其他两个节点是在centos7.2镜像上面构建的Docker容器，安装了mysql5.7和keepalived软件。

在Docker启动容器后执行systemctl命令遇到的坑记录一下。

解决 CentOS7 容器 `Failed to get D-Bus connection: Operation not permitted`

首先要先在后台启动一个 CentOS7 容器（注意不要少参数）：

````bash
docker run -d -e "container=docker" --privileged=true -v /sys/fs/cgroup:/sys/fs/cgroup --name node001 centos7.2-mysql /usr/sbin/init
````

然后再进入这个容器即可：

```bash
docker exec -it node001 /bin/bash
```

## 安装必要软件

```bash
yum install -y net-tools iproute
```

## 安装MySQL双主结构

安装mysql5.7的步骤请参考我的另一篇 [《MySQL核心知识整理》](https://www.xncoding.com/2016/09/17/database/mysql.html)

安装好之后要对mysql进行配置。

重要提示：创建好数据库后，请再创建业务数据库访问用户，只给有限的权限。不要使用super权限的用户比如root来进行业务访问。

### 修改配置文件

master1中有关复制的配置如下，`vim /etc/my.cnf`

```
[mysqld]
# master-master cluster settings --added by Xiong Neng
log-bin=mysql-bin
server-id=1
log_slave_updates=1
secure_file_priv=""
```

master2中相关配置如下，`vim /etc/my.cnf`

```
[mysqld]
# master-master cluster settings --added by Xiong Neng
log-bin=mysql-bin
server-id=2
log_slave_updates=1
read_only=1
```

### 创建复制用户

先登录mysql：

```bash
mysql -u root -p
```

master1中创建：

```
mysql> CREATE USER 'repl'@'172.17.0.3' IDENTIFIED BY '_vsMysql0';

# 查一下加密后的密码
mysql> select password('_vsMysql0');
+-------------------------------------------+
| password('_vsMysql0')                     |
+-------------------------------------------+
| *225F5795D77E27308BB5158C81B21A1023DEACDB |
+-------------------------------------------+
1 row in set, 1 warning (0.00 sec)

mysql> GRANT REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'repl'@'172.17.0.3' IDENTIFIED BY PASSWORD '*225F5795D77E27308BB5158C81B21A1023DEACDB';
mysql> flush privileges;
mysql> show master status;
+------------------+----------+--------------+------------------+-------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------------+----------+--------------+------------------+-------------------+
| mysql-bin.000001 |     4364 |              |                  |                   |
+------------------+----------+--------------+------------------+-------------------+
# 请记住上面的File和Position，在另外一个节点上面配置master的时候需要用到
```

如果创建的用户输错了，就把用户drop掉：

```
use mysql
mysql> select user,host from user;
mysql> drop user 'repl'@'172.17.0.3';
```

master2中创建：

```
mysql> CREATE USER 'repl'@'172.17.0.2' IDENTIFIED BY '_vsMysql0';
mysql> GRANT REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'repl'@'172.17.0.2' IDENTIFIED BY PASSWORD '*225F5795D77E27308BB5158C81B21A1023DEACDB';
mysql> flush privileges;
mysql> show master status;
+------------------+----------+--------------+------------------+-------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------------+----------+--------------+------------------+-------------------+
| mysql-bin.000001 |     2259 |              |                  |                   |
+------------------+----------+--------------+------------------+-------------------+
# 请记住上面的File和Position，在另外一个节点上面配置master的时候需要用到
```

### 执行CHANGE MASTER TO语句

因是从头搭建MySQL主从复制集群，所以不需要获取全局读锁来得到二进制日志文件的位置，直接根据`show master status`的输出来确认。

master1上执行：

```
mysql> CHANGE MASTER TO
    -> MASTER_HOST='172.17.0.3',
    -> MASTER_USER='repl',
    -> MASTER_PASSWORD='_vsMysql0',
    -> MASTER_LOG_FILE='mysql-bin.000001',
    -> MASTER_LOG_POS=2259;
```

master2上执行：

```
mysql> CHANGE MASTER TO
    -> MASTER_HOST='172.17.0.2',
    -> MASTER_USER='repl',
    -> MASTER_PASSWORD='_vsMysql0',
    -> MASTER_LOG_FILE='mysql-bin.000001',
    -> MASTER_LOG_POS=4364;
```

### 执行start slave语句

分别在两个节点上执行：

```
mysql> start slave;
```

如果出现下列错误：

```
ERROR 1872 (HY000): Slave failed to initialize relay log info structure from the repository
```

执行：

```
mysql> reset slave;
mysql> start slave;
```

然后执行下面语句查看复制是否搭建成功：

```
show slave status\G
```

成功标准：

```
Slave_IO_Running: Yes
Slave_SQL_Running: Yes
```

## 配置Keepalived

安装Keepalived：

```
yum install -y rsyslog
yum install -y keepalived
```

### master1的配置

在master1上面，修改Keepalived的配置文件 `vim /etc/keepalived/keepalived.conf`：

除了`global_defs`部分定义外，其余都删了，然后添加下面的：

```
vrrp_script chk_mysql {
    script "/etc/keepalived/check_mysql.sh"
    interval 10   #设置检查间隔时长，可根据自己的需求自行设定
}

vrrp_instance VI_1 {
    state BACKUP  #通过下面的priority来区分MASTER和BACKUP，也只有如此，底下的nopreempt才有效
    interface eth0@if49
    virtual_router_id 51
    priority 100
    advert_int 1
    nopreempt     #防止切换到从库后，主keepalived恢复后自动切换回主库
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    track_script {
        chk_mysql
    }

    virtual_ipaddress {
        172.17.0.4/16
    }
}
```

要注意的几个就是，`interface eth0`这个是指定HA监测网络的接口，在docker上面执行`ifconfig`
可以查看到网络接口名字，`172.17.0.4/16`是虚拟IP地址。

关于keepalived的参数的详细介绍，可参考：[LVS+Keepalived搭建高可用负载均衡集群](http://www.cnblogs.com/ivictor/p/5261445.html)

其中，`/etc/keepalived/check_mysql.sh`内容如下：

```bash
#!/bin/bash

###判断如果上次检查的脚本还没执行完，则退出此次执行
if [ `ps -ef|grep -w "$0"|grep -v "grep"|wc -l` -gt 2 ];then
    exit 0
fi
mysql_con='mysql -uroot -p_vsBaoBao122'
error_log="/etc/keepalived/check_mysql.err"

###定义一个简单判断mysql是否可用的函数
function excute_query {
    ${mysql_con} -e "select 1;" 2>> ${error_log}
}

###定义无法执行查询，且mysql服务异常时的处理函数
function service_error {
    echo -e "`date "+%F  %H:%M:%S"`    -----mysql service error, now stop keepalived-----" >> ${error_log}
    systemctl stop keepalived.service &>> ${error_log}
    echo "DB1 keepalived has stoped"|mail -s "DB1 keepalived stoped" xn@enzhico.com 2>> ${error_log}
    echo -e "\n---------------------------------------------------------\n" >> ${error_log}
}

###定义无法执行查询,但mysql服务正常的处理函数
function query_error {
    echo -e "`date "+%F  %H:%M:%S"`    -----query error, but mysql service ok, retry after 15s-----" >> ${error_log}
    sleep 15
    excute_query
    if [ $? -ne 0 ];then
        echo -e "`date "+%F  %H:%M:%S"`    -----still can't execute query-----" >> ${error_log}
        
        ###对DB1设置read_only属性
        echo -e "`date "+%F  %H:%M:%S"`    -----set read_only = 1 on DB1-----" >> ${error_log}
        mysql_con -e "set global read_only = 1;" 2>> ${error_log}
        
        ###kill掉当前客户端连接
        echo -e "`date "+%F  %H:%M:%S"`    -----kill current client thread-----" >> ${error_log}
        rm -f /tmp/kill.sql &>/dev/null
        ###这里其实是一个批量kill线程的小技巧
        mysql_con -e 'select concat("kill ",id,";") from  information_schema.PROCESSLIST where command="Query" or command="Execute" into outfile "/tmp/kill.sql";'
        mysql_con -e "source /tmp/kill.sql"
        sleep 2  ###给kill一个执行和缓冲时间
        echo -e "`date "+%F  %H:%M:%S"`    -----stop keepalived-----" >> ${error_log}
        systemctl stop keepalived.service &>> ${error_log}
        echo "DB1 keepalived has stoped"|mail -s "DB1 keepalived has stoped" xn@enzhico.com 2>> ${error_log}
        echo -e "\n---------------------------------------------------------\n" >> ${error_log}
    else
        echo -e "`date "+%F  %H:%M:%S"`    -----query ok after 15s-----" >> ${error_log}
        echo -e "\n---------------------------------------------------------\n" >> ${error_log}
    fi
}

###检查开始: 执行查询
excute_query
if [ $? -ne 0 ];then
    systemctl status mysqld.service &>/dev/null
    if [ $? -ne 0 ];then
        service_error
    else
        query_error
    fi
fi

```

通过具体的查询语句来判断数据库服务的可用性，如果查询失败，则判断mysqld进程本身的状态，如果不正常，则直接停止当前节点的keepalived，
将VIP转移到另外一个节点，如果正常，则等待15s，再次执行查询语句，还是失败，则将当前的master节点设置为read_only，
并kill掉当前的客户端连接，然后停止当前的keepalived。

### master2的配置

在master2上面，修改Keepalived的配置文件 `vim /etc/keepalived/keepalived.conf`：

除了`global_defs`部分定义外，其余都删了，然后添加下面的：

```
vrrp_instance VI_1 {
    state BACKUP
    interface eth0@if51
    virtual_router_id 51
    priority 90
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    notify_master /etc/keepalived/notify_master_mysql.sh
    virtual_ipaddress {
        172.17.0.4/16
    }
}
```

其中，`/etc/keepalived/notify_master_mysql.sh`的内容如下：

```bash
#!/bin/bash
###当keepalived监测到本机转为MASTER状态时，执行该脚本

change_log=/etc/keepalived/state_change.log
mysql_con='mysql -uroot -p_vsBaoBao122'
echo -e "`date "+%F  %H:%M:%S"`   -----keepalived change to MASTER-----" >> $change_log

slave_info() {
    ###统一定义一个函数取得slave的position、running、和log_file等信息
    ###根据函数后面所跟参数来决定取得哪些数据
    if [ $1 = slave_status ];then
        slave_stat=`${mysql_con} -e "show slave status\G;"|egrep -w "Slave_IO_Running|Slave_SQL_Running"`
        Slave_IO_Running=`echo $slave_stat|awk '{print $2}'`
        Slave_SQL_Running=`echo $slave_stat|awk '{print $4}'`
    elif [ $1 = log_file -a $2 = pos ];then
        log_file_pos=`${mysql_con} -e "show slave status\G;"|egrep -w "Master_Log_File|Read_Master_Log_Pos|Relay_Master_Log_File|Exec_Master_Log_Pos"`
        Master_Log_File=`echo $log_file_pos|awk '{print $2}'`
        Read_Master_Log_Pos=`echo $log_file_pos|awk '{print $4}'`
        Relay_Master_Log_File=`echo $log_file_pos|awk '{print $6}'`
        Exec_Master_Log_Pos=`echo $log_file_pos|awk '{print $8}'`
    fi
}

action() {
    ###经判断'应该&可以'切换时执行的动作
    echo -e "`date "+%F  %H:%M:%S"`    -----set read_only = 0 on DB2-----" >> $change_log
    
    ###解除read_only属性
    ${mysql_con} -e "set global read_only = 0;" 2>> $change_log
    
    echo "DB2 keepalived change to MASTER"|mail -s "DB2 keepalived change to MASTER" xn@enzhico.com 2>> $change_log
    echo -e "---------------------------------------------------------\n" >> $change_log
}

slave_info slave_status
if [ $Slave_SQL_Running = Yes ];then
    i=0  ###一个计数器
    slave_info log_file pos
    ###判断从master接收到的binlog是否全部在本地执行(这样仍无法完全确定从库已追上主库，因为无法完全保证io_thread没有延时(由网络传输问题导致的从库落后的概率很小)
    until [ $Master_Log_File = $Relay_Master_Log_File -a $Read_Master_Log_Pos = $Exec_Master_Log_Pos ]
     do
        if [ $i -lt 10 ];then   ###将等待exec_pos追上read_pos的时间限制为10s
            ###输出消息到日志，等待exec_pos=read_pos
            echo -e "`date "+%F  %H:%M:%S"`  -----Relay_Master_Log_File=$Relay_Master_Log_File,Exec_Master_Log_Pos=$Exec_Master_Log_Pos is behind Master_Log_File=$Master_Log_File,Read_Master_Log_Pos=$Read_Master_Log_Pos, wait......" >> $change_log
            i=$(($i+1))
            sleep 1
            slave_info log_file pos
        else
            echo -e "The waits time is more than 10s,now force change. Master_Log_File=$Master_Log_File Read_Master_Log_Pos=$Read_Master_Log_Pos Relay_Master_Log_File=$Relay_Master_Log_File Exec_Master_Log_Pos=$Exec_Master_Log_Pos" >> $change_log
            action
            exit 0
        fi
    done
    action
else
    slave_info log_file pos
    echo -e "DB2's slave status is wrong,now force change. Master_Log_File=$Master_Log_File Read_Master_Log_Pos=$Read_Master_Log_Pos Relay_Master_Log_File=$Relay_Master_Log_File Exec_Master_Log_Pos=$Exec_Master_Log_Pos" >> $change_log
    action
fi
```

整个脚本的逻辑是让从的Exec_Master_Log_Pos尽可能的追上Read_Master_Log_Pos，它给了10s的限制，如果还是没有追上，
则直接将master2设置为主（通过解除read_only属性），其实这里面还是有待商榷的，譬如10s的限制是否合理，
还是一定需要`Exec_Master_Log_Pos=Read_Master_Log_Pos`才切换。

## 恢复原主

当原主恢复正常后，如何将VIP从master2切回到master1中呢？

自己写一个脚本，放到master2主机上面，手动执行将主库切换回DB1的操作，比如将其命名为`/etc/keepalived/reset_master_mysql.sh`
，如下：

```bash
#!/bin/bash
###手动执行将主库切换回DB1的操作

root_pwd="_vsBaoBao122"
repl_pwd="_vsMysql0"
master1_ip="172.17.0.2"
change_log=/etc/keepalived/state_change.log

mysql_con="mysql -uroot -p${root_pwd}"

echo -e "`date "+%F  %H:%M:%S"`    -----change to BACKUP manually-----" >> $change_log
echo -e "`date "+%F  %H:%M:%S"`    -----set read_only = 1 on DB2-----" >> $change_log
$mysql_con -e "set global read_only = 1;" 2>> $change_log

###kill掉当前客户端连接
echo -e "`date "+%F  %H:%M:%S"`    -----kill current client thread-----" >> $change_log
rm -f /tmp/kill.sql &>/dev/null
###这里其实是一个批量kill线程的小技巧
$mysql_con -e 'select concat("kill ",id,";") from  information_schema.PROCESSLIST where command="Query" or command="Execute" into outfile "/tmp/kill.sql";'
$mysql_con -e "source /tmp/kill.sql" 2>> $change_log
sleep 2  ###给kill一个执行和缓冲时间

###确保DB1已经追上了,下面的repl为复制所用的账户，-h后跟DB1的内网IP
log_file_pos=`mysql -urepl -p${repl_pwd} -h${master1_ip} -e "show slave status\G;"|egrep -w "Master_Log_File|Read_Master_Log_Pos|Relay_Master_Log_File|Exec_Master_Log_Pos"`
Master_Log_File=`echo $log_file_pos|awk '{print $2}'`
Read_Master_Log_Pos=`echo $log_file_pos|awk '{print $4}'`
Relay_Master_Log_File=`echo $log_file_pos|awk '{print $6}'`
Exec_Master_Log_Pos=`echo $log_file_pos|awk '{print $8}'`
until [ $Read_Master_Log_Pos = $Exec_Master_Log_Pos -a $Master_Log_File = $Relay_Master_Log_File ]
do
    echo -e "`date "+%F  %H:%M:%S"`  -----DB1 Exec_Master_Log_Pos($exec_pos) is behind Read_Master_Log_Pos($read_pos), wait......" >> $change_log
    sleep 1
    log_file_pos=`mysql -urepl -p${repl_pwd} -h${master1_ip} -e "show slave status\G;"|egrep -w "Master_Log_File|Read_Master_Log_Pos|Relay_Master_Log_File|Exec_Master_Log_Pos"`
    Master_Log_File=`echo $log_file_pos|awk '{print $2}'`
    Read_Master_Log_Pos=`echo $log_file_pos|awk '{print $4}'`
    Relay_Master_Log_File=`echo $log_file_pos|awk '{print $6}'`
    Exec_Master_Log_Pos=`echo $log_file_pos|awk '{print $8}'`
done

###然后解除DB1的read_only属性
echo -e "`date "+%F  %H:%M:%S"`    -----set read_only = 0 on DB1-----" >> $change_log
ssh ${master1_ip} 'mysql -uroot -p_vsBaoBao122 -e "set global read_only = 0;" && systemctl restart keepalived.service' 2>> $change_log

###重启DB2的keepalived使VIP漂移到DB1
echo -e "`date "+%F  %H:%M:%S"`    -----make VIP move to DB1-----" >> $change_log
systemctl restart keepalived.service &>> $change_log

echo "DB2 keepalived to BACKUP, master db to DB1"|mail -s "DB2 keepalived change to BACKUP" xn@enzhico.com 2>> $change_log

echo -e "--------------------------------------------------\n" >> $change_log

```

注：涉及到切换主备，就会有中断时间，所以推荐此步骤在业务低谷期执行。

## 测试

集群配置完成后

### 正常情况

keepalived日志：

master1

![](https://xnstatic-1253397658.file.myqcloud.com/mysql101.png)

master2

![](https://xnstatic-1253397658.file.myqcloud.com/mysql103.png)

----------

mysql的read_only变量：

master1

![](https://xnstatic-1253397658.file.myqcloud.com/mysql102.png)

master2

![](https://xnstatic-1253397658.file.myqcloud.com/mysql103.png)

注意：read_only=1只读模式，可以限定普通用户进行数据修改的操作，但不会限定具有super权限的用户的数据修改操作；

----------

IP地址：

master1

![](https://xnstatic-1253397658.file.myqcloud.com/mysql105.png)

master2

![](https://xnstatic-1253397658.file.myqcloud.com/mysql106.png)

通过查看上面三个，很明显就知道VIP现在在master1上面，并且master1上面的数据库是主库可写，
master2是从库只读。

--------------------------

接下来我测试一下数据库的操作，看看数据是否能同步到从库上面。

master1

![](https://xnstatic-1253397658.file.myqcloud.com/mysql107.png)

master2

![](https://xnstatic-1253397658.file.myqcloud.com/mysql108.png)

可以看到，无论是在master1上面创建数据库、新建表、插入新数据，在master2上面都能同步到。

### 异常情况

这里通过将master1上面的mysql服务停掉来模拟异常情况。

```
systemctl stop mysqld.service
```

keepalived日志：

master1

![](https://xnstatic-1253397658.file.myqcloud.com/mysql109.png)

master2

![](https://xnstatic-1253397658.file.myqcloud.com/mysql110.png)

----------

mysql的read_only变量：

master2

![](https://xnstatic-1253397658.file.myqcloud.com/mysql113.png)

可以看到master2此时的read_only关闭，可写了。

---------

IP地址：

master1

![](https://xnstatic-1253397658.file.myqcloud.com/mysql111.png)

master2

![](https://xnstatic-1253397658.file.myqcloud.com/mysql112.png)

VIP现在跑到master2上面了。

------------

通过VIP 172.17.0.4访问数据库，并在t_test表里面删除一条数据。

![](https://xnstatic-1253397658.file.myqcloud.com/mysql114.png)

### 主库恢复

当我修复好主库后，要重新还原回来，这里修复完成后就重启mysql服务，

master1上面：

```
systemctl restart mysqld.service
```

master2上面执行：

```
/etc/keepalived/reset_master_mysql.sh
```

在master1上面查看ip：
![](https://xnstatic-1253397658.file.myqcloud.com/mysql115.png)

在看看master1中的数据库，刚刚我删的一条数据同步回来没？

![](https://xnstatic-1253397658.file.myqcloud.com/mysql116.png)

其他的我就不再截图演示了，总之成功切换回来了，并且数据库也同步回来了。

## 重要说明

三个脚本文件：

1. master1上面的`/etc/keepalived/check_mysql.sh`
2. master2上面的`/etc/keepalived/notify_master_mysql.sh`
3. master2上面的`/etc/keepalived/reset_master_mysql.sh`

另外需要配置master2的root用户可以通过ssh免密码登录master1

两个容器上面都要执行：

```bash
passwd
yum -y install openssh-server openssh-clients
ssh-keygen -t rsa
```

然后再master2上面执行：

```
ssh-copy-id 172.17.0.2
```

## 自定义keepalived的日志输出文件

keepalived的日志默认是输出到/var/log/messages中，这样不便于查看。如何自定义keepalived的日志输出文件呢？

修改`/etc/sysconfig/keepalived`文件

```
KEEPALIVED_OPTIONS="-D -d -S 0"
```

修改`/etc/rsyslog.conf`：

```
# keepalived -S 0 
local0.*                                                /var/log/keepalived.log
```

重启syslog：

```
systemctl restart rsyslog
systemctl restart keepalived
```

## FAQ

keepalived运行后产生大量的core.*文件，撑爆磁盘空间，查看日志`/var/log/keepalived.log`发现报错：

```
IPVS: Can't initialize ipvs: Protocol not available
```

解决办法：

首先你必须得先安装ipvsadm：

```
yum install -y ipvsadm
```

要是你已经安装过了,这句就直接忽略掉吧,之后在执行一下命令：

```
ipvsadm
```

还是报错：

```
Can't initialize ipvs: Protocol not available
Are you sure that IP Virtual Server is built in the kernel or as module?
```

原因是ip_vs模块系统默认没有自动加载，可以通过

```
lsmod | grep ip_vs
```

查看一下，如果没有任何输出则表示ip_vs模块并没有被内核加载，需要手动加载：

```
modprobe ip_vs
modprobe ip_vs_wrr
```

如果要让系统开机加载此模块的话得讲刚才那两句话写到`/etc/rc.local`文件中，这样开机就能自动加载了。

这样试过仍然还是不行，再想办法。

执行

```
modinfo ip_vs
```

发现没有这个ip_vs模块可以加载。

1. 首先必须要保证宿主机上面安装`yum install -y ipvsadm && ipvsadm`
2. 启动docker容器必须加参数`--privileged`

重新启动docker容器就正常了。

## 改进方案

在Keepalived中有两种模式，分别是master->backup模式和backup->backup模式，在master->backup模式下，一旦主库宕掉，
虚拟IP会自动漂移到从库，当主库修复后，keepalived启动后，还会把虚拟IP抢过来，即使你设置nopreempt（不抢占）的方式抢占IP的动作也会发生。
在backup->backup模式下，当主库宕掉后虚拟IP会自动漂移到从库上，当原主恢复之后重启keepalived服务，并不会抢占新主的虚拟IP，
即使是优先级高于从库的优先级别，也不会抢占IP。

为了减少IP的漂移次数，生产中我们通常是把修复好的主库当做新主库的备库，本篇的方案还是通过手动执行`reset_master_mysql.sh`
将master1切换回主库。
更优方案是master1修复好之后，只需要启动上面的keepalived软件。master2作为新的主库，而master1现在变成了备库。

另外当主库出现问题的时候通知机制，现在并没有配置邮箱服务，后面把邮件通知服务也加上去就完美了。

有空的时候把这个改进方案补充上来。

