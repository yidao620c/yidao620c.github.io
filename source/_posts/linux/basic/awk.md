---
title: awk命令笔记
date: 2016-09-22 22:22:22 +0800
toc: true
categories: [ linux ]
tags: [ linux ]
---

awk是一种样式扫描与处理工具。但其功能却大大强于sed和grep。awk提供了极其强大的功能，它几乎可以完成grep和sed所能完成的全部工作，
同时，它还可以可以进行样式装入、流控制、数学运算符、进程控制语句甚至于内置的变量和函数。
它具备了一个完整的语言所应具有的几乎所有精美特性。实际上，awk的确拥有自己的语言：样式扫描和处理语言。
它允许您创建简短的程序，这些程序读取输入文件、为数据排序、处理数据、对输入执行计算以及生成报表，还有无数其他的功能。

简单来说awk就是把文件逐行的读入，以空格为默认分隔符将每行切片，切开的部分再进行各种分析处理。
<!-- more -->

## 基本用法

从`netstat -tnlp > netstat.txt`命令中提取了如下信息作为用例：

```
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name
tcp        0      0 0.0.0.0:8778            0.0.0.0:*               LISTEN      2689/python
tcp        0      0 0.0.0.0:3306            0.0.0.0:*               LISTEN      2265/mysqld
tcp        0      0 0.0.0.0:139             0.0.0.0:*               LISTEN      18777/smbd
tcp        0      0 127.0.0.1:35437         0.0.0.0:*               LISTEN      23842/python2
tcp        0      0 0.0.0.0:80              0.0.0.0:*               LISTEN      1916/nginx: master
tcp        0      0 0.0.0.0:4369            0.0.0.0:*               LISTEN      1704/epmd
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN      973/sshd
tcp        0      0 127.0.0.1:58205         0.0.0.0:*               LISTEN      23681/python2
tcp        0      0 0.0.0.0:445             0.0.0.0:*               LISTEN      18777/smbd
tcp        0      0 0.0.0.0:33443           0.0.0.0:*               LISTEN      13976/phantomjs
tcp        0      0 0.0.0.0:8008            0.0.0.0:*               LISTEN      2548/unicorn_rails
tcp        0      0 192.168.217.161:5673    0.0.0.0:*               LISTEN      951/beam.smp
tcp        0      0 0.0.0.0:25673           0.0.0.0:*               LISTEN      951/beam.smp
tcp6       0      0 :::139                  :::*                    LISTEN      18777/smbd
tcp6       0      0 :::4369                 :::*                    LISTEN      1704/epmd
tcp6       0      0 :::22                   :::*                    LISTEN      973/sshd
tcp6       0      0 :::3128                 :::*                    LISTEN      2268/(squid-1)
tcp6       0      0 :::1017                 :::*                    LISTEN      988/xnServer
tcp6       0      0 :::443                  :::*                    LISTEN      950/httpd
tcp6       0      0 :::445                  :::*                    LISTEN      18777/smbd
```

下面是最简单最常用的awk示例，比如要想输出第1列和第4列：

```bash
$ awk '{print $1, $4}' netstat.txt
```

我们再来看看awk的格式化输出，和C语言的printf没什么两样：

```bash
$ awk '{printf "%-6s %-6s %-6s %-22s %-16s %-12s\n",$1,$2,$3,$4,$5,$6}' netstat.txt
Proto  Recv-Q Send-Q Local                  Address          Foreign
tcp    0      0      0.0.0.0:8778           0.0.0.0:*        LISTEN
tcp    0      0      0.0.0.0:3306           0.0.0.0:*        LISTEN
tcp    0      0      0.0.0.0:139            0.0.0.0:*        LISTEN
tcp    0      0      127.0.0.1:35437        0.0.0.0:*        LISTEN
tcp    0      0      0.0.0.0:80             0.0.0.0:*        LISTEN
tcp    0      0      0.0.0.0:4369           0.0.0.0:*        LISTEN
...
```

## 内置变量

awk经常会使用到一些内置变量，下面是它们的含义：

| 内置变量     | 含义                                     |
|----------|----------------------------------------|
| $0       | 当前记录（这个变量中存放着整个行的内容）                   |
| $1~$n    | 当前记录的第n个字段，字段间由FS分隔                    |
| FS       | 输入字段分隔符，默认是空格或Tab                      |
| NF       | 当前记录中的字段个数，就是有多少列                      |
| NR       | 已经读出的记录数，就是行号，从1开始，如果有多个文件话，这个值也是不断累加中 |
| FNR      | 当前记录数，与NR不同的是，这个值会是各个文件自己的行号           |
| RS       | 输入的记录分隔符，默认为换行符                        |
| OFS      | 输出字段分隔符，默认也是空格                         |
| ORS      | 输出的记录分隔符，默认为换行符                        |
| FILENAME | 当前输入文件的名字                              |

## 过滤记录

如何过滤记录（下面过滤条件为：第三列的值为0 && 第7列包含字符串'python'）

```bash
$ awk '$3==0 && $7 ~ /python/' netstat.txt
tcp        0      0 0.0.0.0:8778            0.0.0.0:*    LISTEN      2689/python
tcp        0      0 127.0.0.1:35437         0.0.0.0:*    LISTEN      23842/python2
tcp        0      0 127.0.0.1:58205         0.0.0.0:*    LISTEN      23681/python2
```

其中的"=="为比较运算符，其他比较运算符：!=, >, <, >=, <=

还有"~"表示正则匹配，这个就相当强大了。

如果我们需要表头的话，我们可以引入内建变量NR：

```bash
$ awk '$3==0 && $6=="LISTEN" || NR==1 ' netstat.txt
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name
tcp        0      0 0.0.0.0:8778            0.0.0.0:*               LISTEN      2689/python
tcp        0      0 0.0.0.0:3306            0.0.0.0:*               LISTEN      2265/mysqld
tcp        0      0 0.0.0.0:139             0.0.0.0:*               LISTEN      18777/smbd
tcp        0      0 127.0.0.1:35437         0.0.0.0:*               LISTEN      23842/python2
tcp        0      0 0.0.0.0:80              0.0.0.0:*               LISTEN      1916/nginx: master
tcp        0      0 0.0.0.0:4369            0.0.0.0:*               LISTEN      1704/epmd
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN      973/sshd
tcp        0      0 127.0.0.1:58205         0.0.0.0:*               LISTEN      23681/python2
...
```

再加上格式化输出：

```bash
$ awk '$3==0 && $6=="LISTEN" || NR==1 {printf "%-20s %-20s %s\n",$4,$5,$6}' netstat.txt
Local                Address              Foreign
0.0.0.0:8778         0.0.0.0:*            LISTEN
0.0.0.0:3306         0.0.0.0:*            LISTEN
0.0.0.0:139          0.0.0.0:*            LISTEN
127.0.0.1:35437      0.0.0.0:*            LISTEN
0.0.0.0:80           0.0.0.0:*            LISTEN
0.0.0.0:4369         0.0.0.0:*            LISTEN
0.0.0.0:22           0.0.0.0:*            LISTEN
...
```

## 指定分隔符

```bash
$ awk 'BEGIN{FS=":"} {print $1,$3,$6}' /etc/passwd
root 0 /root
bin 1 /bin
daemon 2 /sbin
adm 3 /var/adm
lp 4 /var/spool/lpd
sync 5 /sbin
shutdown 6 /sbin
halt 7 /sbin
mail 8 /var/spool/mail
...
```

上面的命令也等价于：（-F的意思就是指定分隔符）

```bash
$ awk -F: '{print $1,$3,$6}' /etc/passwd
```

注：如果你要指定多个分隔符，你可以这样来：

```bash
awk -F '[;:]'
```

再来看一个以\t作为分隔符输出的例子（下面使用了/etc/passwd文件，这个文件是以:分隔的）：

```bash
$ awk -F: '{print $1,$3,$6}' OFS="\t" /etc/passwd
root    0    /root
bin     1    /bin
daemon  2    /sbin
adm     3    /var/adm
lp      4    /var/spool/lpd
sync    5    /sbin
```

## 字符串匹配

```bash
$ awk '$7 ~ /python/ || NR==1 {print NR,$4,$5,$6}' OFS="\t" netstat.txt
1	Local	Address	Foreign
2	0.0.0.0:8778	0.0.0.0:*	LISTEN
5	127.0.0.1:35437	0.0.0.0:*	LISTEN
9	127.0.0.1:58205	0.0.0.0:*	LISTEN
```

上面示例第7列匹配"python"字符串，其实 ~ 表示模式开始。/ /中是模式，这就是一个正则表达式的匹配。

其实awk可以像grep一样的去匹配一整行，就像这样：

```bash
$ awk '/python/' netstat.txt
tcp   0      0 0.0.0.0:8778     0.0.0.0:*     LISTEN      2689/python
tcp   0      0 127.0.0.1:35437  0.0.0.0:*     LISTEN      23842/python2
tcp   0      0 127.0.0.1:58205  0.0.0.0:*     LISTEN      23681/python2
```

再来看看模式取反的例子:

```bash
awk '!/python/' netstat.txt
```

## 统计

下面的命令计算所有的.py文件和.txt文件大小总和:

```bash
$ ls -l *.py *.txt | awk '{sum+=$5} END {print sum}'
3369
```
