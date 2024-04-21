---
title: centos6上安装Hadoop和HBase
date: 2015-05-27 14:01:57 +0800
toc: true
categories: [大数据]
tags: [hadoop]
---

操作系统：CentOS 6.5 64位

在linux环境安装Hadoop之前，我们需要使用到ssh，所以要先安装ssh，并且创建一个hadoop用户

**备注：** 下面所有的命令中，以#开头的表示是root用户，以$开头的是普通用户

安装SSH，先切换到root用户，执行下列步骤
```bash
rpm -qa |grep ssh  #检查是否装了SSH包
yum install openssh-server  #安装ssh
chkconfig --list sshd #检查SSHD是否设置为开机启动
chkconfig --level 2345 sshd on  #如果没设置启动就设置下.
service sshd restart  #重新启动
```
<!-- more -->

### 创建hadoop用户
```bash
$ su
password:
# useradd hadoop
# passwd hadoop
New passwd:
Retype new passwd
```

### 生成pub-key
切换到hadoop用户后，执行
```bash
$ ssh-keygen -t rsa
$ cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
$ chmod 0600 ~/.ssh/authorized_keys
```
然后确认下是否能正常使用ssh连接
```bash
ssh localhost
```

### 安装JDK1.7

进入oracle官网<http://www.oracle.com/technetwork/java/javase/downloads/jdk7-downloads-1880260.html>

下载jdk-7u79-linux-x64.gz，然后执行：
```bash
$ tar zxf jdk-7u79-linux-x64.gz
$ ls
jdk1.7.0_79 jdk-7u79-linux-x64.gz
$ su
password:
# mv jdk1.7.0_79 /usr/local/
# exit
```
打开~/.bashrc文件，写入JAVA_HOME环境变量
```bash
export JAVA_HOME=/usr/local/jdk1.7.0_79
export PATH= $PATH:$JAVA_HOME/bin
```
保存刷新下：`$ source ~/.bashrc`

切换到root用户，然后执行下面的语句确保JDK版本更改完成
```
# alternatives --install /usr/bin/java java /usr/local/jdk1.7.0_79/bin/java 2
# alternatives --install /usr/bin/javac javac /usr/local/jdk1.7.0_79/bin/javac 2
# alternatives --install /usr/bin/jar jar /usr/local/jdk1.7.0_79/bin/jar 2
# alternatives --set java /usr/local/jdk1.7.0_79/bin/java
# alternatives --set javac /usr/local/jdk1.7.0_79/bin/javac
# alternatives --set jar /usr/local/jdk1.7.0_79/bin/jar
```
最后执行下：`java -version`看看是不是已经成功安装了JDK7

### 安装配置Hadoop

#### 下载Hadoophadoop2.6.0

下载地址：<http://apache.fayea.com/hadoop/common/stable/hadoop-2.6.0.tar.gz>

```
$ su
password:
# cd /usr/local
# wget http://apache.fayea.com/hadoop/common/stable/hadoop-2.6.0.tar.gz
# tar xzf hadoop-2.6.0.tar.gz
# mv hadoop-2.6.0 hadoop
# exit
```
hadoop有很多种模式，本篇我们演示的是伪分布式模式，包括后面的HBase也选择这种模式。

#### 配置Hadoop环境
第一步，配置环境变量

打开~/.bashrc文件，写入如下内容
```bash
export HADOOP_HOME=/usr/local/hadoop
export HADOOP_MAPRED_HOME=$HADOOP_HOME
export HADOOP_COMMON_HOME=$HADOOP_HOME
export HADOOP_HDFS_HOME=$HADOOP_HOME
export YARN_HOME=$HADOOP_HOME
export HADOOP_COMMON_LIB_NATIVE_DIR=$HADOOP_HOME/lib/native
export PATH=$PATH:$HADOOP_HOME/sbin:$HADOOP_HOME/bin
export HADOOP_INSTALL=$HADOOP_HOME
```
然后应用设置
```bash
$ source ~/.bashrc
```
第二步，hadoop配置文件

hadoop的配置文件都放在"$HADOOP_HOME/etc/hadoop"目录中，
你可以根据自己的需要来修改它们。

在此之前，还需要修改下hadoop-env.sh，更改其中的JAVA_HOME变量
```bash
vim /usr/local/hadoop/etc/hadoop/hadoop-env.sh
```
然后修改JAVA_HOME为真实的目录
```bash
export JAVA_HOME=/usr/local/jdk1.7.0_79
```
接下来我们去到hadoop的配置文件目录
```
$ cd $HADOOP_HOME/etc/hadoop
```bash
1\. 首先打开core-site.xml，写入如下配置

```xml
<configuration>
    <property>
        <name>hadoop.tmp.dir</name>
        <value>/usr/local/hadoop/tmp</value>
    </property>
    <property>
        <name>fs.defaultFS</name>
        <value>hdfs://localhost:9000</value>
    </property>
</configuration>
```

2\. 然后打开hdfs-site.xml，写入如下配置

```xml
<configuration>
    <property>
        <name>dfs.replication</name>
        <value>1</value>
    </property>
    <property>
        <name>dfs.name.dir</name>
        <value>file:///home/hadoop/hadoopinfra/hdfs/namenode</value>
    </property>
    <property>
        <name>dfs.data.dir</name>
        <value>file:///home/hadoop/hadoopinfra/hdfs/datanode</value>
    </property>
    <property>
        <name>dfs.permissions</name>
        <value>false</value>
    </property>
</configuration>
```

上面的文件夹需要我们手动来创建，那么我们创建下就行了
```bash
$ mkdir -p /home/hadoop/hadoopinfra/hdfs/namenode
$ mkdir -p /home/hadoop/hadoopinfra/hdfs/datanode
```

3\. 然后打开yarn-site.xml文件，写入如下配置

```xml
<configuration>
    <property>
        <name>yarn.nodemanager.aux-services</name>
        <value>mapreduce_shuffle</value>
    </property>
    <property>
        <name>yarn.resourcemanager.scheduler.address</name>
        <value>localhost:54313</value>
    </property>
</configuration>
```

4\. 配置mapred-site.xml，先重命名
```bash
$ cp mapred-site.xml.template mapred-site.xml
```
打开mapred-site.xml文件，写入如下配置

```xml
<configuration>
    <property>
        <name>yarn.nodemanager.aux-services</name>
        <value>mapreduce_shuffle</value>
    </property>
</configuration>
```

#### 确认Hadoop的安装
1\. NameNode确认
```
$ cd ~
$ hdfs namenode -format
```
结果应该类似下面
```
STARTUP_MSG: Starting NameNode
STARTUP_MSG:   host = centos00/127.0.0.1
STARTUP_MSG:   args = [-format]
STARTUP_MSG:   version = 2.6.0
...
/************************************************************
SHUTDOWN_MSG: Shutting down NameNode at centos00/127.0.0.1
************************************************************/
```

2\. Hadoop dfs确认
```
$ start-dfs.sh
```
结果应该类似下面
```
Starting namenodes on [localhost]
localhost: starting namenode, logging to ....out
localhost: starting datanode, logging to ....out
Starting secondary namenodes [0.0.0.0]
The authenticity of host '0.0.0.0 (0.0.0.0)' can't be established.
RSA key fingerprint is fd:01:fc:f2:53:a0:58:8e:96:9c:5f:f2:6e:5b:69:1a.
Are you sure you want to continue connecting (yes/no)? yes
0.0.0.0: Warning: Permanently added '0.0.0.0' (RSA) to the list of known hosts.
0.0.0.0: starting secondarynamenode, logging to ...
```

3\. Yarn Srcipt确认
```
$ start-yarn.sh
```
结果应该类似下面这样
```
starting yarn daemons
starting resourcemanager, logging to ....out
localhost: starting nodemanager, logging to ....out
```

4\. 浏览器访问Hadoop

默认访问Hadoop的端口是50070，在浏览器中打开链接<http://localhost:50070>来访问Hadoop服务。

5\. 浏览器确认应用集群

默认访问应用集群的端口号是8088，在浏览器中打开链接<http://localhost:8088>来确认下。

### 安装HBase
你可以在三种模式下安装HBase：单机模式、伪分布式模式、全分布式模式。
下面我们演示在伪分布式模式下HBase的安装和配置。

#### 下载HBase
```
$ su
# cd /usr/local/
# wget http://apache.fayea.com/hbase/hbase-0.98.12/hbase-0.98.12-hadoop2-bin.tar.gz
# tar -zxvf hbase-0.98.12-hadoop2-bin.tar.gz
# mv hbase-0.98.12-hadoop2 hbase
# chown -R hadoop:hadoop /usr/local/hbase
```

#### 配置hbase-site.xml
```
su hadoop
$ cd /usr/local/hbase/conf
```
然后打开hbase-env.sh文件，修改如下
```
export JAVA_HOME=/usr/local/jdk1.7.0_79
```
修改hbase-site.xml文件，如下

```xml
<property>
    <name>hbase.rootdir</name>
    <value>hdfs://localhost:9000/hbase</value>
</property>

<property>
    <name>hbase.zookeeper.property.dataDir</name>
    <value>/home/hadoop/zookeeper</value>
</property>

<property>
    <name>hbase.cluster.distributed</name>
    <value>true</value>
</property>
```
编辑/etc/profile，增加HBASE_HOME环境变量
```
export HBASE_HOME=/usr/local/hbase
export PATH=$PATH:$HBASE_HOME/bin
```
应用更改。
```
source /etc/profile
```
OK，现在为止，HBase的安装和配置都已经完成了。

现在你可以通过执行start-hbase.sh来启动HBase
```
$ cd /usr/local/hbase/bin
$ ./start-hbase.sh
```
然后执行`jps`命令应该可以看到HMaster和HRegionServer这两个进程。类似下面
```
10941 DataNode
13744 HQuorumPeer
14207 Jps
11126 SecondaryNameNode
11276 ResourceManager
10840 NameNode
13843 HMaster
10016 HRegionServer
11378 NodeManager
```

如果没有看到，可以查看日志`/usr/local/hbase/logs/hbase-hadoop-master-xx.log`

#### 在HDFS中检查HBase目录
HBase会在HDFS中创建自己的目录，在hadoop目录下面执行：
```
$ ./bin/hadoop fs -ls /hbase
```
显示如下
```
drwxr-xr-x   - hadoop supergroup          0 2015-04-24 16:06 /hbase/.tmp
drwxr-xr-x   - hadoop supergroup          0 2015-04-24 16:06 /hbase/WALs
drwxr-xr-x   - hadoop supergroup          0 2015-04-24 16:06 /hbase/data
-rw-r--r--   1 hadoop supergroup         42 2015-04-24 16:06 /hbase/hbase.id
-rw-r--r--   1 hadoop supergroup          7 2015-04-24 16:06 /hbase/hbase.version
drwxr-xr-x   - hadoop supergroup          0 2015-04-24 16:06 /hbase/oldWALs
```

那么恭喜你，配置成功了！
