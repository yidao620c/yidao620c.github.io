---
layout: post
title: JVM性能分析工具jstack介绍
date: 2018-06-25 23:19:22 +0800
toc: true
categories: java
tags: java
abbrlink: 3719
---

JDK本身提供了很多方便的JVM性能调优监控工具，除了集成式的VisualVM和jConsole外，
还有jps、jstack、jmap、jhat、jstat、hprof等小巧的工具，每一种工具都有其自身的特点，
用户可以根据你需要检测的应用或者程序片段的状况，适当的选择相应的工具进行检测，
先通过一个表格形式简要介绍下这几个命令的作用和使用方法。本文重点介绍jstack的使用方法。

命令	     | 作用
---------|---------------------------------------
jps	     | 基础工具
jstack	 | 查看某个Java进程内的线程堆栈信息
jmap	 | jmap导出堆内存，然后使用jhat来进行分析
jhat	 | jmap导出堆内存，然后使用jhat来进行分析
jstat	 | JVM统计监测工具
hprof	 | hprof能够展现CPU使用率，统计堆内存使用情况
<!-- more -->

## jps使用

可以列出本机所有java进程的pid

选项

* -q 仅输出VM标识符，不包括class name,jar name,arguments in main method
* -m 输出main method的参数
* -l 输出完全的包名，应用主类名，jar的完全路径名
* -v 输出jvm参数
* -V 输出通过flag文件传递到JVM中的参数(.hotspotrc文件或-XX:Flags=所指定的文件
* -Joption 传递参数到vm,例如:-J-Xms48m

示例：

```bash
[root@CZT-FS1 board-api]# jps -lvm
67136 board-api-1.0.0-SNAPSHOT.jar --spring.config.location=application.yml -Xms1024m -Xmx1024m
100547 board-web-1.0.0-SNAPSHOT.jar --spring.config.location=application.yml -Xms512m
8819 app-manage-api-1.0.0-SNAPSHOT.jar --spring.config.location=application.yml -Xms512m
120226 adm-web-1.0.0-SNAPSHOT.jar --spring.config.location=file:./application.yml -Xms512m
105268 wechat-api-1.0.0-SNAPSHOT.jar --spring.config.location=application.yml -Xms512m
89172 sun.tools.jps.Jps -lvm -Denv.class.path=.:/usr/local/jdk/lib/dt.jar:/usr/local/jdk/lib/tools.jar:/usr/local/jdk/jre/lib -Dapplication.home=/usr/local/jdk -Xms8m
44921 app-manage-1.0.0-SNAPSHOT.jar --spring.config.location=file:./application.yml -Xms512m -Xmx1024m
```

我们选取PID=67136的Java进程作为后续研究对象。

## top使用

除了常用的打印所有进程使用资源外，还可以对单独的进程，打印线程资源排行榜，按T键可对TIME倒序排列，
也就是CPU运行时间。TIME列就是各个Java线程耗费的CPU时间，我们线程pid为67163的线程作为后续线程研究对象

```bash
[root@CZT-FS1 board-api]# top -Hp 67136
top - 11:22:26 up 166 days, 17:06,  1 user,  load average: 0.07, 0.02, 0.00
Tasks: 140 total,   0 running, 140 sleeping,   0 stopped,   0 zombie
Cpu(s):  0.1%us,  0.0%sy,  0.0%ni, 99.9%id,  0.0%wa,  0.0%hi,  0.0%si,  0.0%st
Mem:  16309360k total, 16120904k used,   188456k free,   174440k buffers
Swap:  8175612k total,   461868k used,  7713744k free,  7831512k cached

   PID USER      PR  NI  VIRT  RES  SHR S %CPU %MEM    TIME+  COMMAND
 67163 root      20   0 6825m 867m  14m S  2.0  5.4  16:22.87 java
 67136 root      20   0 6825m 867m  14m S  0.0  5.4   0:00.00 java
 67137 root      20   0 6825m 867m  14m S  0.0  5.4   0:10.82 java
 67138 root      20   0 6825m 867m  14m S  0.0  5.4   0:00.32 java
 67139 root      20   0 6825m 867m  14m S  0.0  5.4   0:00.32 java
 67140 root      20   0 6825m 867m  14m S  0.0  5.4   0:00.31 java
 67141 root      20   0 6825m 867m  14m S  0.0  5.4   0:00.35 java
 67142 root      20   0 6825m 867m  14m S  0.0  5.4   0:00.35 java
 67143 root      20   0 6825m 867m  14m S  0.0  5.4   0:00.36 java
 67144 root      20   0 6825m 867m  14m S  0.0  5.4   0:00.33 java
 67145 root      20   0 6825m 867m  14m S  0.0  5.4   0:00.31 java
 67146 root      20   0 6825m 867m  14m S  0.0  5.4   0:08.29 java
 67147 root      20   0 6825m 867m  14m S  0.0  5.4   0:00.02 java
 67148 root      20   0 6825m 867m  14m S  0.0  5.4   0:00.35 java
 67149 root      20   0 6825m 867m  14m S  0.0  5.4   0:00.00 java
 67150 root      20   0 6825m 867m  14m S  0.0  5.4   0:25.08 java
```

## jstack使用

jstack主要用来查看某个Java进程内的线程堆栈信息。语法格式如下：

```bash
jstack [option] pid
```

参数如下：

1. -l	long listings，会打印出额外的锁信息，在发生死锁时可以用jstack -l pid来观察锁持有情况
2. -m	mixed mode，不仅会输出Java堆栈信息，还会输出C/C++堆栈信息（比如Native方法）

```bash
[root@CZT-FS1 board-api]# jstack -l 67136 | more
Full thread dump Java HotSpot(TM) 64-Bit Server VM (25.161-b12 mixed mode):

"Thread-12126" #12389 daemon prio=6 os_prio=0 tid=0x00007f3190075800 nid=0x15d08 runnable [0x00007f31f43c4000]
   java.lang.Thread.State: RUNNABLE
	at java.net.SocketInputStream.socketRead0(Native Method)
	at java.net.SocketInputStream.socketRead(SocketInputStream.java:116)
	at java.net.SocketInputStream.read(SocketInputStream.java:171)
	at java.net.SocketInputStream.read(SocketInputStream.java:141)
	at java.io.BufferedInputStream.read1(BufferedInputStream.java:284)
	at java.io.BufferedInputStream.read(BufferedInputStream.java:345)
	- locked <0x00000000f49e3158> (a java.io.BufferedInputStream)
	at java.io.BufferedInputStream.fill(BufferedInputStream.java:246)
	at java.io.BufferedInputStream.read(BufferedInputStream.java:265)
	- locked <0x00000000f49e51a8> (a org.apache.commons.net.telnet.TelnetInputStream)
	at org.apache.commons.net.telnet.TelnetInputStream.__read(TelnetInputStream.java:132)
	at org.apache.commons.net.telnet.TelnetInputStream.run(TelnetInputStream.java:603)
	at java.lang.Thread.run(Thread.java:748)

   Locked ownable synchronizers:
	- None

"Thread-12125" #12388 daemon prio=6 os_prio=0 tid=0x00007f31c4602000 nid=0x15cf2 runnable [0x00007f31f4bcc000]
   java.lang.Thread.State: RUNNABLE
	at java.net.SocketInputStream.socketRead0(Native Method)
	at java.net.SocketInputStream.socketRead(SocketInputStream.java:116)
	at java.net.SocketInputStream.read(SocketInputStream.java:171)

```

使用`printf "%x\n"`，获得线程ID=的十六进制值。

```bash
[root@CZT-FS1 board-api]# printf "%x\n" 67163
1065b
```

查看该线程的堆栈：

```bash
[root@CZT-FS1 board-api]# jstack -l 67136 | grep 1065b -A20
"System Clock" #17 daemon prio=5 os_prio=0 tid=0x00007f322d089000 nid=0x1065b runnable [0x00007f320487c000]
   java.lang.Thread.State: TIMED_WAITING (parking)
	at sun.misc.Unsafe.park(Native Method)
	- parking to wait for  <0x00000000c19c8d18> (a java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject)
	at java.util.concurrent.locks.LockSupport.parkNanos(LockSupport.java:215)
	at java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject.awaitNanos(AbstractQueuedSynchronizer.java:2078)
	at java.util.concurrent.ScheduledThreadPoolExecutor$DelayedWorkQueue.take(ScheduledThreadPoolExecutor.java:1093)
	at java.util.concurrent.ScheduledThreadPoolExecutor$DelayedWorkQueue.take(ScheduledThreadPoolExecutor.java:809)
	at java.util.concurrent.ThreadPoolExecutor.getTask(ThreadPoolExecutor.java:1074)
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1134)
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624)
	at java.lang.Thread.run(Thread.java:748)

   Locked ownable synchronizers:
	- None

"Druid-ConnectionPool-Destroy-1390913202" #16 daemon prio=5 os_prio=0 tid=0x00007f322ea27800 nid=0x1065a waiting on condition [0x00007f320497d000]
   java.lang.Thread.State: TIMED_WAITING (sleeping)
	at java.lang.Thread.sleep(Native Method)
	at com.alibaba.druid.pool.DruidDataSource$DestroyConnectionThread.run(DruidDataSource.java:2538)
```
