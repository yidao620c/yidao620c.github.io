---
title: JVM工具-基础监控工具
date: 2021-04-27 20:15:23 +0800
toc: true
categories: [ java ]
tags: [ jvm ]
---

JDK本身提供了很多方便的JVM性能调优监控工具，除了集成式的VisualVM和JConsole外，还有jps、jstack、jmap、jhat、jstat、hprof等小巧的工具，
每一种工具都有其自身的特点， 用户可以根据你需要检测的应用或者程序片段的状况，适当的选择相应的工具进行检测，
先通过一个表格形式简要介绍下这几个命令的作用和使用方法。

命令         | 作用
------------|---------------------------------------
jps         | JVM进程ID查询工具
jstat       | JVM统计信息监测工具
jstack      | 查看某个Java进程内的线程堆栈信息
jmap        | jmap导出堆内存，然后使用jhat来进行分析
jhat        | jmap导出堆内存，然后使用jhat来进行分析
hprof       | hprof能够展现CPU使用率，统计堆内存使用情况

<!-- more -->

## jps：JVM进程ID查询工具

可以列出本机所有java进程的pid，很多工具的输入依赖这个pid。

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
105268 wechat-api-1.0.0-SNAPSHOT.jar --spring.config.location=application.yml -Xms512m
89172 sun.tools.jps.Jps -lvm -Denv.class.path=.:/usr/local/jdk/lib/dt.jar -Dapplication.home=/usr/local/jdk -Xms8m
```

我们选取PID=67136的Java进程作为后续研究对象。

## jstat：JVM统计信息监测工具

jstat用于监视JVM各种运行状态信息，可显示本地或远程（需要开启RMI支持）虚拟机进程中的类加载、内存、垃圾收集、JIT等运行时数据。
在控制台环境下，它是运行期定位虚拟机性能问题的常用工具。

jstat命令格式：

```
jstat [option vmid [interval] [count]]
```

参数interval和count代表查询间隔和次数，如果省略则只查询一次。比如如果要每隔200ms查询一次进程1223垃圾收集状态，一共查询10次。则

```
jstat -gc 1223 200 10
```

jstat主要选项如下

选项               |  作用
------------------|------------------------------------------------------------------------------------
-class            | 监视类加载、卸载数量、总空间以及类装载耗费时间
-gc               | 监视Java堆状况，包括Eden区、2个Suervivor区、老年代、永久代等容量，已用空间，GC合计时间等。
-gccapacity       | 监视内容跟gc基本相同，但主要关注Java堆各个区域使用到的最大、最小空间。
-gcutil           | 监视内容跟gc基本相同，但主要关注已使用空间占总空间百分比。
-gccause          | 监视内容跟gcutil基本相同，但会额外输出导致上一次GC原因。
-gcnew            | 监视新生代GC状态
-gcnewcapacity    | 监视内容跟gcnew基本相同，输出主要关注使用到的最大、最小空间。
-gcold            | 监视老年代GC状态
-gcoldcapacity    | 监视内容跟gcold基本相同，输出主要关注使用到的最大、最小空间。
-gcpermcapacity   | 监视永久代使用到的最大、最小空间。
-compiler         | 输出JIT编译过的方法、耗时信息
-printcompilation | 输出已经被JIT编译的方法

jstat执行样例：

```
jstat -gcutil 12592
S0     S1     E      O      M     CCS    YGC   YGCT    FGC    FGCT    CGC    CGCT     GCT
0.00  92.02   0.00  17.37  97.60  94.52   5    0.027     0    0.000     2    0.007    0.033
```

 缩写   | 含义                              | 原文                                                                            
------|---------------------------------|-------------------------------------------------------------------------------
 S0   | 新生代中Survivor space 0区已使用空间的百分比	 | Survivor space 0 utilization as a percentage of the space's current capacity. 
 S1   | 新生代中Survivor space 1区已使用空间的百分比	 | Survivor space 1 utilization as a percentage of the space's current capacity. 
 E    | 新生代已使用空间百分比                     | Eden space utilization as a percentage of the space's current capacity        
 O    | 老年代已使用空间百分比                     | Old space utilization as a percentage of the space's current capacity.        
 M    | 元空间                             | Metaspace utilization as a percentage of the space's current capacity         
 CCS  | 压缩类空间利用率百分比                     | Compressed class space utilization as a percentage                            
 YGC  | YGC事件的数量                        | Number of young generation GC events.                                         
 YGCT | 年轻一代垃圾收集时间                      | Young generation garbage collection time                                      
 FGC  | FGC事件的数量                        | Number of full GC events.                                                     
 FGCT | 完全垃圾收集时间                        | Full garbage collection time                                                  
 CGC  | 并发GC统计                          | Concurrent GC Count                                                           
 CGCT | 并发GC收集时间                        | Concurrent GC Collection Time                                                 
 GCT  | 垃圾回收总时间                         | Total garbage collection time.                                                

CGC和CGCT是ZGC的标志，ZGC（The Z Garbage Collector）是JDK 11中推出的一款低延迟垃圾回收器，是一个并发垃圾回收器。

## jmap：Java内存映像工具

jmap命令用于生成堆转储快照，一般称为heapdump或dump文件，该工具在JDK9中集成到了JHSDB中。

命令格式：`jmap [option] vmid`

option的几个主要选项如下

选项               |  作用
------------------|----------------------------------------------------------------------------------
-dump             | 生成dump文件，格式为-dump:[live,]format=b,file=<filename>，其中live表示是否只dump存活对象
-finalizerinfo    | 显示在F-Queue中等待finalizer线程执行finalize方法的对象
-heap             | 显示堆详细信息，比如使用的回收器、参数配置、分代情况
-histo            | 显示堆中对象统计信息，包括类、实例数量、合计容量
-pemstat          | 以ClassLoader为统计口径显示永久代内存状态
-F                | 强制生成dump快照，这个在-dump选项没有响应时使用

使用jmap的样例

```
C:\Users\xiongneng>jmap -dump:format=b,file=test.dump 12592
Dumping heap to C:\Users\xiongneng\test.dump ...
Heap dump file created [29413292 bytes in 0.089 secs]
```

## top使用

除了常用的打印所有进程使用资源外，还可以对单独的进程，打印线程资源排行榜，按T键可对TIME倒序排列，也就是CPU运行时间。
TIME列就是各个Java线程耗费的CPU时间，我们线程pid为67163的线程作为后续线程研究对象

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
 67150 root      20   0 6825m 867m  14m S  0.0  5.4   0:25.08 java
```

## jstack：线程堆栈跟踪工具

jstack主要用来查看某个Java进程内的线程堆栈信息，生成的文件一般称为threaddump或javacore文件。
线程快照就是当前虚拟机内每一条线程正在执行的方法堆栈集合，生成线程快照目的通常是定位线程出现停顿原因。
比如线程死锁、死循环、请求外部资源导致长时间挂起等。

该工具在JDK9中已集成到了JHSDB中。

语法格式如下：

```bash
jstack [option] vmid
```

选项            |  作用
---------------|-------------------------------------------
-F             | 正常输出的请求不被响应时，强制输出线程堆栈
-l             | 除了堆栈外，还会输出关于锁的附加信息
-m             | 如果调用本地方法，可显示C/C++堆栈信息

```bash
[root@CZT-FS1 board-api]# jstack -l 67136 | more
Full thread dump Java HotSpot(TM) 64-Bit Server VM (25.161-b12 mixed mode):

"Thread-12126" #12389 daemon prio=6 os_prio=0 tid=0x00007f3190075800 nid=0x15d08 runnable [0x00007f31f43c4000]
   java.lang.Thread.State: RUNNABLE
    at java.net.SocketInputStream.socketRead0(Native Method)
    - locked <0x00000000f49e3158> (a java.io.BufferedInputStream)
    at java.io.BufferedInputStream.fill(BufferedInputStream.java:246)
    at java.io.BufferedInputStream.read(BufferedInputStream.java:265)
    - locked <0x00000000f49e51a8> (a org.apache.commons.net.telnet.TelnetInputStream)
    at org.apache.commons.net.telnet.TelnetInputStream.__read(TelnetInputStream.java:132)
    at org.apache.commons.net.telnet.TelnetInputStream.run(TelnetInputStream.java:603)
    at java.lang.Thread.run(Thread.java:748)

   Locked ownable synchronizers:
    - None
```

如果想要查看某个线程的堆栈，先使用上面的top命令查看到线程ID号，然后使用`printf "%x\n"`，获得该线程ID的十六进制值。

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
    at java.lang.Thread.run(Thread.java:748)

   Locked ownable synchronizers:
    - None
```

