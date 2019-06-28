---
title: ceph pool相关命令
date: '2017-03-16 12:07:09 +0800'
comments: true
toc: true
categories: ceph
tags:
  - ceph
abbrlink: 51292
---

安装完ceph集群后默认会创建一个名字叫rbd的存储池，可自己创建存储池。

pool是ceph存储数据时的逻辑分区，它起到namespace的作用。
每个pool包含一定数量的PG，PG里的对象被映射到不同的OSD上，因此pool是分布到整个集群的。

有两种类型的pool，一种是副本（replicated pools），一种是纠删码（erasure code pools）。
副本池是我们最常用的类型。

pool也支持snapshot功能。可以运行ceph osd pool mksnap命令创建pool的快照，并且在必要的时候恢复它。
还可以设置pool的拥有者属性，从而进行访问控制。

下面记录一下和pool相关的命令

## 查询pool列表
``` bash
ceph osd lspools
```

## 创建pool

创建pool之前可更改两个pg默认配置：
```
osd pool default pg num = 100
osd pool default pgp num = 100
```

pg_num是指定创建的pg的个数，会有一组编号，然后pgp_num就是控制pg到osd的映射分布。一般最好将pgp_num设置成一样。

``` bash
ceph osd pool create {pool-name} {pg-num} [{pgp-num}] [replicated] [crush-ruleset-name] [expected-num-objects]
ceph osd pool create testpool 128 128 replicated
```

## 删除存储池

``` bash
ceph osd pool delete {pool-name} [{pool-name} --yes-i-really-really-mean-it]
```

如果你单独为这个pool创建了crush_ruleset，那么最好也将这个也删了。
``` bash
# 查看pool的ruleset id
crush_ruleset=$(ceph osd pool get {pool-name} crush_ruleset)
# 比如找到了crush_ruleset名字为123
pool_num=$(ceph osd dump | grep "^pool" | grep "crush_ruleset ${crush_ruleset} " | wc -l)
# 如果pool_num=1，则表示没有其他存储池使用到，就先找到rule_name
rule_name=$(ceph osd crush dump | grep -A1 '"rule_id": 123' | tail -n 1 | awk '{print $2}' | awk -F\" '{print $2}')
# 删除crush_ruleset
ceph osd crush rule rm $rule_name
# 查看crush map
ceph osd crush dump
z=/tmp/z && ceph osd getcrushmap -o $z && crushtool -d $z -o $z && cat $z && rm -f $z
```

## 重命名存储池
``` bash
ceph osd pool rename {current-pool-name} {new-pool-name}
```

## 快照
``` bash
# 创建快照
ceph osd pool mksnap {pool-name} {snap-name}
# 删除快照
ceph osd pool rmsnap {pool-name} {snap-name}
# rbd命令操作快照
rbd -p <pool name> snap create {snap-name}     Create a snapshot.
rbd -p <pool name> snap list                   Dump list of image snapshots.
rbd -p <pool name> snap protect {snap-name}    Prevent a snapshot from being deleted.
rbd -p <pool name> snap purge                  Deletes all snapshots.
rbd -p <pool name> snap remove {snap-name}     Deletes a snapshot.
rbd -p <pool name> snap rename  {snap} {snap1} Rename a snapshot.
rbd -p <pool name> snap rollback {snap-name}   Rollback image to snapshot.
rbd -p <pool name> snap unprotect {snap-name}  Allow a snapshot to be deleted.
```

## 获取/设置存储池属性
``` bash
ceph osd pool get {pool-name} {key}
ceph osd pool set {pool-name} {key} {value}
```

列出一些比较常见的属性：

* size           - 副本数
* pg_num         - pg数量
* pgp_num        - pg放置数量
* crush_ruleset  - 规则id

## 获取pool的空间使用情况

```
[root@node001 ~]# ceph df
GLOBAL:
    SIZE     AVAIL     RAW USED     %RAW USED
    449G      449G         195M          0.04
POOLS:
    NAME     ID     USED     %USED     MAX AVAIL     OBJECTS
    rbd      0         0         0          269G           0
    sb       18        0         0             0           0
    hb       19      250         0        92071M           3
```


