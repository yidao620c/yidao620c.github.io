---
title: tcpdump抓包
date: 2020-06-05 06:15:15 +0800
toc: true
categories: [ 开发工具 ]
tags: [ 网络工具 ]
---

比较常见的抓包方法是使用tcpdump在linux机器上运行，生成pcap文件。 然后拖到windows机器，下载wireshark来可视化分析。

常用操作

```bash
./tcpdump -i eth0 -vv -s0 -weth0.pcap
# 特定IP地址，不管是源IP还是目的IP
./tcpdump host 10.3.19.129 -i eth0 -vv -s0 -weth0.pcap
./tcpdump  dst host 10.3.19.129 -i eth0 -vv -s0 -w eth0 .pcap
# and操作
tcpdump tcp port 22 and src host 123.207.116.169
# or操作
tcpdump src host 123.207.116.168 or src host 123.207.116.169
# 特定源IP地址
tcpdump src host 18.16.202.169
# 特定目标地址
tcpdump dst host 18.16.202.169
# 监听特定主机之间的通信
tcpdump ip host 210.27.48.1 and 210.27.48.2
# 监听特定端口
tcpdump port 8083 -vv
# 监听tcp协议，并加数据包写入abc.cap
tcpdump tcp port 8083 -w  ./abc.cap
```

<!-- more -->

稍微复杂例子

```bash
tcpdump tcp -i eth1 -t -s 0 -c 100 and dst port ! 22 and src net 192.168.1.0/24 -w ./target.cap
```

参数说明

```
tcp: ip icmp arp rarp 和 tcp、udp、icmp这些选项等都要放到第一个参数的位置，用来过滤数据报的类型
-i eth1 : 只抓经过接口eth1的包
-t : 不显示时间戳
-s 0 : 抓取数据包时默认抓取长度为68字节。加上-S 0 后可以抓到完整的数据包
-c 100 : 只抓取100个数据包
dst port ! 22 : 不抓取目标端口是22的数据包
src net 192.168.1.0/24 : 数据包的源网络地址为192.168.1.0/24
-w ./target.cap : 保存成cap文件，方便用ethereal(即wireshark)分析
```