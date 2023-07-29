---
title: Linux磁盘分区总结
date: 2017-03-14 15:32:07 +0800
toc: true
categories: [ linux ]
tags: [ linux ]
---

最近在做ceph的日志盘分区时候遇到一些问题记录下来。比较古老的分区工具是使用fdisk，后来因为磁盘越来越大，
而fdisk遇到分区容量超过2TB时无能为力，并且分区还要限制最多4个主分区。越来越不适用现代磁盘，
后来有了parted工具专门用来对大磁盘分区的。

硬盘分区最常见的类型为`msdos`和`gpt`，前者表示MBR分区，而后者表示GPT分区。

传统的BIOS只支持MBR分区硬盘启动，一个硬盘只能分成四个分区，并且单个分区最大不超过2TB。
EFI支持GPT分区启动的，GPT分区没有分区数目的限制并且单个分区可以超过2TB。
<!-- more -->

> ***MBR***<br/>
> MBR分区表(即主引导记录)大家都很熟悉。所支持的最大卷为2T，而且最多4个主分区或3个主分区加一个扩展分区
>
> ***GPT***<br/>
> GPT（即GUID分区表）是源自EFI标准的一种较新的磁盘分区表结构的标准，是未来磁盘分区的主要形式。
> 与MBR分区方式相比，突破MBR4个主分区限制，每个磁盘最多支持128个分区，持大于2T的分区，最大卷可达18EB。

使用parted命令查看磁盘分区类型:

```bash
[root@node200 ~]# parted /dev/sdb print
Model: SMC SMC2108 (scsi)
Disk /dev/sdb: 999GB
Sector size (logical/physical): 512B/512B
Partition Table: gpt
Disk Flags:

Number  Start   End     Size    File system  Name          Flags
 2      1049kB  10.7GB  10.7GB               ceph journal
 1      10.7GB  999GB   988GB   xfs          ceph data
```

输出结果中的`Partition Table: gpt`表示gpt分区

这里顺带解释下物理扇区和逻辑扇区的概念，上面的`Sector size (logical/physical): 512B/512B`，
也就是硬盘的每个扇区实际大小是512B，不过我们做分区都是针对逻辑扇区的，也就是logical为单位，
这里也是512B，一般来讲磁盘会将一个物理扇区划分成多个逻辑扇区，逻辑扇区才是分区最小单位。

## fdisk使用

fdisk用来创建MBR分区，这个工具应该都会用了，这里只是简单介绍几个。

显示所有硬盘分区情况

```bash
fdisk -l
```

显示具体单个硬盘分区情况

```bash
fdisk -l /dev/sdb
```

交互显示方便提示fdisk的所有命令如何使用

```bash
fdisk /dev/sdb
```

进入交互模式后按m可列出所有命令:

```bash
[root@node200 ~]# fdisk /dev/sdb
Welcome to fdisk (util-linux 2.23.2).

Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.


Command (m for help): m
Command action
   d   delete a partition
   g   create a new empty GPT partition table
   G   create an IRIX (SGI) partition table
   l   list known partition types
   m   print this menu
   n   add a new partition
   o   create a new empty DOS partition table
   q   quit without saving changes
   s   create a new empty Sun disklabel
   w   write table to disk and exit
```

## parted使用

参考：https://www.gnu.org/software/parted/manual/parted.html

Parted 命令分为两种模式：命令行模式和交互模式。

1. 命令行模式: `parted [option] device [command]`, 比较适合编程应用。
2. 交互模式:  `parted [option] device` 类似于使用`fdisk /dev/xxx`

parted是一个可以分区并进行分区调整的工具，他可以创建、删除、移动、复制、
调整ext2 Linux-swap fat fat32 reiserfs类型的分区，
可以创建、调整、移动Macintosh的HFS分区，检测jfs，ntfs，ufs，xfs分区。

### 显示分区信息

```bash
parted list  # 列出所有磁盘分区信息
parted /dev/sdb print # 打印出sdb的分区信息
blkid  # 查看分区名
```

### 创建分区类型

```bash
parted /dev/sdb mklabel label-type
```

label-type可以是："bsd", "dvh", "gpt",  "loop","mac", "msdos", "pc98", or "sun"。
一般的pc机都是msdos格式，如果分区大于2T则需要选用gpt格式的分区表。

### 分区转换

转换成GPT:

```bash
parted /dev/sdb mklabel gpt
```

转换成MBR:

```bash
parted /dev/sdb mklabel msdos
```

### 创建分区

```bash
parted /dev/sdb mkpart [part-type fs-type name] start end
```

创建一个part-type类型的分区，part-type可以是："primary", "logical", or "extended"，
只有在分区类型为'msdos'或'dvh'的时候才能指定这个参数。

如果指定fs-type则在创建分区的同时进行格式化，start和end指的是分区的起始位置，单位默认是M。

### 设置分区名

```bash
parted /dev/sdb name 1 "name"
```

这种设置只能用在Mac, PC98 和 GPT类型的分区表，设置时名字用引号括起来

### 调整分区大小

```bash
parted /dev/sdb resize 1 start end
```

### 删除一个分区

```bash
parted /dev/sdb rm 1
```

### 移动分区

```bash
parted /dev/sdb move 1 start end
```

## sgdisk使用

参考：http://www.rodsbooks.com/gdisk/sgdisk-walkthrough.html

与fdisk创建MBR分区一样，sgdisk是一个创建GPT分区的工具，它也有交互模式gdisk和命令行模式sgdisk两种，
一般手动分区就选择gdisk会保险一点，这里我讲的是命令行模式sgdisk，方便编程实现。

### 查看所有GPT分区

先可执行parted命令查看磁盘分区类型:

```bash
parted /dev/sdb print
```

然后对于gpt类型的就可以用sgdisk命令查看了:

```bash
[root@node200 ~]# sgdisk -p /dev/sdb
Disk /dev/sdb: 1951170560 sectors, 930.4 GiB
Logical sector size: 512 bytes
Disk identifier (GUID): EFA3DAF2-5AF2-44A0-9AC7-D9F50E60D831
Partition table holds up to 128 entries
First usable sector is 34, last usable sector is 1951170526
Partitions will be aligned on 2048-sector boundaries
Total free space is 4061 sectors (2.0 MiB)

Number  Start (sector)    End (sector)  Size       Code  Name
   1        20973568      1951170526   920.4 GiB   FFFF  ceph data
   2            2048        20971520   10.0 GiB    FFFF  ceph journal
```

### 查看某个分区

```bash
[root@node200 ~]# sgdisk --info=1 /dev/sdb
Partition GUID code: 4FBD7E29-9D25-41B8-AFD0-062C0CEFF05D (Unknown)
Partition unique GUID: 4382DDCA-69AE-4753-B181-E800EAA62ADD
First sector: 20973568 (at 10.0 GiB)
Last sector: 1951170526 (at 930.4 GiB)
Partition size: 1930196959 sectors (920.4 GiB)
Attribute flags: 0000000000000000
Partition name: 'ceph data'
```

### 删除磁盘所有分区

```bash
[root@node200 ~]# sgdisk --zap-all --clear --mbrtogpt -g /dev/sdd
GPT data structures destroyed! You may now partition the disk using fdisk or
other utilities.
The operation has completed successfully.
```

### 创建分区

创建某个分区前，请先查看这个磁盘分区表，查看扇区大小sector，然后算好起始扇区start和结束扇区end。

比如我要创建第1个分区/dev/sdd1，大小1GiB，扇区数量为`1*1024*1024*1024/512=2097152`。
起始扇区start=1，结束扇区end=2097152，type code为8300，分区名为"ceph journal"

```bash
sgdisk -n 1:1:2097152 -t 1:8300 -c 1:"ceph journal" -g -p /dev/sdd
```

type code一般就是8300(Linux filesystem)，可通过`sgdisk -L`来查看所有。

出错:

```
Could not create partition 1 from 1 to 2097152
Could not change partition 1's type code to 8300!
Disk /dev/sdd: 16777216 sectors, 8.0 GiB
Logical sector size: 512 bytes
Disk identifier (GUID): C2CD3A11-3C49-4DD7-B5E9-8DA6F980CA81
Partition table holds up to 128 entries
First usable sector is 34, last usable sector is 16777182
Partitions will be aligned on 2048-sector boundaries
Total free space is 16777149 sectors (8.0 GiB)

Number  Start (sector)    End (sector)  Size       Code  Name
Error encountered; not saving changes.
```

仔细看看发现`First usable sector is 34`，第一个可用扇区是34（前面33个扇区不能用），
所以得往后移33再次执行:

```bash
[root@node200 ~]# sgdisk -n 1:34:2097185 -t 1:8300 -p /dev/sdd
Information: Moved requested sector from 34 to 2048 in
order to align on 2048-sector boundaries.
Disk /dev/sdd: 16777216 sectors, 8.0 GiB
Logical sector size: 512 bytes
Disk identifier (GUID): C2CD3A11-3C49-4DD7-B5E9-8DA6F980CA81
Partition table holds up to 128 entries
First usable sector is 34, last usable sector is 16777182
Partitions will be aligned on 2048-sector boundaries
Total free space is 14682011 sectors (7.0 GiB)

Number  Start (sector)    End (sector)  Size       Code  Name
   1            2048         2097185   1023.0 MiB  8300
The operation has completed successfully.
```

执行成功！

不过你可用看到就是start扇区变成了2048，原因是需要分区对齐
`Moved requested sector from 34 to 2048 in order to align on 2048-sector boundaries`

如果忘记设置分区名了，就用下面命令:

```bash
sgdisk -c 1:"ceph journal" /dev/sdd
```

查看下是否成功:

```bash
[root@node200 ~]# sgdisk -p /dev/sdd
Disk /dev/sdd: 16777216 sectors, 8.0 GiB
Logical sector size: 512 bytes
Disk identifier (GUID): C2CD3A11-3C49-4DD7-B5E9-8DA6F980CA81
Partition table holds up to 128 entries
First usable sector is 34, last usable sector is 16777182
Partitions will be aligned on 2048-sector boundaries
Total free space is 14682011 sectors (7.0 GiB)

Number  Start (sector)    End (sector)  Size       Code  Name
   1            2048         2097185   1023.0 MiB  8300  ceph journal
```

## 分区格式化

一般来讲分区的时候不要做格式化，分完区之后单独来做格式化。

```bash
mkfs -t xfs /dev/sdd1
```

## 分区挂载

显示系统已挂载的分区列表:

```bash
df -h
```

挂载命令格式: `mount [-t fstype] [-o options] device dir`

```
1. -t vfstype 指定文件系统的类型，通常不必指定。mount 会自动选择正确的类型。常用类型有：
　　光盘或光盘镜像：iso9660
　　DOS fat16文件系统：msdos
　　Windows 9x fat32文件系统：vfat
　　Windows NT ntfs文件系统：ntfs
　　Mount Windows文件网络共享：smbfs
　　UNIX(LINUX) 文件网络共享：nfs
2. -o options 主要用来描述设备或档案的挂接方式。常用的参数有：
　　loop：用来把一个文件当成硬盘分区挂接上系统
　　ro：采用只读方式挂接设备
　　rw：采用读写方式挂接设备
　　iocharset：指定访问文件系统所用字符集
3. device 要挂接(mount)的设备。
4. dir设备在系统上的挂接点(mount point)。
```

实际例子:

```bash
mount /dev/sdd1 /opt/winstore/var/log
```

另外还有两个命令也有用

查看所有磁盘以及相应的分区挂载:

```bash
lsblk
```

查看系统所有分区名:

```bash
blkid
```

