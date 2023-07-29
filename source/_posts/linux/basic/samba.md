---
title: CentOS7通过samba共享文件夹
date: 2018-02-20 19:09:22 +0800
toc: true
categories: [ linux ]
tags: [ linux ]
---

Samba是在Linux系统上实现SMB协议的一个免费软件，由服务器及客户端程序构成。
SMB(Server Messages Block, 信息服务块)是一种在局域网上共享文件和打印机的一种通信协议，
它为局域网内的不同计算机之间提供文件及打印机等资源的共享服务。
SMB协议是客户机/服务器型协议，客户机通过该协议可以访问服务器上的共享文件系统，打印机及其他资源。

比如我想共享`/home/samba`这个文件夹给其他计算机使用。
<!-- more -->

## 安装samba

```bash
yum install -y samba
```

## 修改samba配置

配置文件是`/etc/samba/smb.conf`

```
# See smb.conf.example for a more detailed config file or
# read the smb.conf manpage.
# Run 'testparm' to verify the config is correct after
# you modified it.

[global]
    workgroup = SAMBA   #samba的工作组，设置成 Windows 的工作组
    security = user     #安全选项，可以是 share|user|server|domain，安全级别递增
    passdb backend = tdbsam #定义口令类型，存在如下3中口令校验
    #smbpasswd：使用smbpasswd命令为系统用户设置Samba服务程序的密码
    #tdbsam：创建数据库文件并使用pdbedit命令建立Samba服务程序的用户
    #ldapsam：基于LDAP服务进行账户验证
    printing = cups
    printcap name = cups
    load printers = yes #设置在Samba服务启动时是否共享打印机设备
    cups options = raw #打印机的选项

[homes]   #共享默认会将用户的主目录共享, 这是不安全的, 可以将其注释
    comment = Home Directories
    valid users = %S, %D%w%S
    browseable = no #指定共享信息是否在"网上邻居"中可见
    writable = yes #定义是否可以执行写入操作，与"read only"相反
    inherit acls = yes

[printers]   #打印机共享
    comment = All Printers
    path = /var/tmp #共享文件的实际路径(重要)
    printable = yes
    create mask = 0600
    browseable = no
    guest ok = no #是否所有人可见，等同于"public"参数

[print$]
    comment = Printer Drivers
    path = /var/lib/samba/drivers
    write list = root
    create mask = 0664
    directory mask = 0775

[rootdir]   #自定义的共享文件夹
    comment = share some files
    path = /home/samba/     #共享的路径
    public = no             #关闭"所有人可见"
    writable = yes          #允许写入操作
```

注意，自己修改时去掉 `#` 后面的备注

## 添加 Samba 用户

创建用于访问共享资源的账户信息。在RHEL 7系统中，Samba服务程序默认使用的是用户口令认证模式（user）。
这种认证模式可以确保仅让有密码且受信任的用户访问共享资源，而且验证过程也十分简单。不过，
只有建立账户信息数据库之后，才能使用用户口令认证模式。
另外，Samba服务程序的数据库要求账户必须在当前系统中已经存在，
否则日后创建文件时将导致文件的权限属性混乱不堪，由此引发错误。

pdbedit命令用于管理SMB服务程序的账户信息数据库，格式为`pdbedit [选项] 账户`。
在第一次把账户信息写入到数据库时需要使用-a参数，以后在执行修改密码、删除账户等操作时就不再需要该参数了。
pdbedit命令中使用的参数以及作用如下所示：

 参数     | 作用          
--------|-------------
 -a 用户名 | 建立Samba用户   
 -x 用户名 | 删除Samba用户   
 -L     | 列出用户列表      
 -Lv    | 列出用户详细信息的列表 

创建samba用户

```bash
[root@master ~]# useradd samba
[root@master ~]# id samba
uid=1001(samba) gid=1001(samba) 组=1001(samba)
[root@master ~]# pdbedit -a -u samba
new password:
retype new password:
Unix username:        samba
NT username:          
Account Flags:        [U          ]
User SID:             S-1-5-21-1309844399-986653456-3519320502-1000
Primary Group SID:    S-1-5-21-1309844399-986653456-3519320502-513
Full Name:            
Home Directory:       \\master\samba
HomeDir Drive:        
Logon Script:         
Profile Path:         \\master\samba\profile
Domain:               MASTER
Account desc:         
Workstations:         
Munged dial:          
Logon time:           0
Logoff time:          三, 06 2月 2036 23:06:39 CST
Kickoff time:         三, 06 2月 2036 23:06:39 CST
Password last set:    二, 25 8月 2020 22:12:06 CST
Password can change:  二, 25 8月 2020 22:12:06 CST
Password must change: never
Last bad password   : 0
Bad password count  : 0
Logon hours         : FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF
```

## 设置SELinux安全上下文

创建用于共享资源的文件目录不仅要考虑到文件读写权限的问题，而且由于/home目录是系统中普通用户的家目录，
因此还需要考虑应用于该目录的SELinux安全上下文所带来的限制。
Samba服务程序配置文件中的注释就有关于SELinux安全上下文策略的说明，
我们只需按照过滤信息中有关SELinux安全上下文策略中的说明中给的值进行修改即可。
修改完毕后执行`restorecon`命令，让应用于目录的新SELinux安全上下文立即生效。

```bash
[root@master ~]# semanage fcontext -a -t samba_share_t /home/samba
[root@master ~]# restorecon -Rv /home/samba
```

设置SELinux服务与策略，使其允许通过Samba服务程序访问普通用户家目录。执行getsebool命令，
筛选出所有与Samba服务程序相关的SELinux域策略，根据策略的名称（和经验）选择出正确的策略条目进行开启即可：

```bash
[root@master ~]# getsebool -a | grep samba
samba_create_home_dirs --> off
samba_domain_controller --> off
samba_enable_home_dirs --> off
samba_export_all_ro --> off
samba_export_all_rw --> off
samba_load_libgfapi --> off
samba_portmapper --> off
samba_run_unconfined --> off
samba_share_fusefs --> off
samba_share_nfs --> off
sanlock_use_samba --> off
tmpreaper_use_samba --> off
use_samba_home_dirs --> off
virt_use_samba --> off
[root@master ~]# setsebool -P samba_enable_home_dirs on
```

## 启动 Samba 服务

启动、停止、查看、开机自启动相关命令。

```bash
systemctl start smb
systemctl stop smb
systemctl status smb
systemctl enable smb
```

同时还要清空防火墙配置：

```bash
iptables -F
systemctl restart network
```

## Windows 访问共享目录

直接 Win + R , 在运行界面输入`\\192.168.1.20`，也就是你的 Linux 主机地址，
会弹出用户名密码输入界面，输入刚刚设置的用户名密码就可以访问。注意这里输入的密码是samba设置的密码，
不是登录Linux系统的密码。因为在RHEL 7系统中，Samba服务程序使用的果然是独立的账户信息数据库。
所以，即便在Linux系统中有一个samba账户，Samba服务程序使用的账户信息数据库中也有一个同名的samba账户，
大家也一定要弄清楚它们各自所对应的密码。此时，我们可以尝试执行查看、写入、更名、删除文件等操作。

如果 Windows 下访问 Linux 下共享目录 , 提示没有权限。

1. 确保 Linux 下防火墙关闭或者是开放共享目录权限
1. 确保 Samba 服务器配置文件 smb.conf 设置没有问题
1. 确保 SELinux配置正确

Samb 还需要开放下面四个端口

```
UDP 137、UDP 138、TCP 139、TCP 445
```

## Linux挂载共享

Samba服务程序还可以实现Linux系统之间的文件共享。现在我们在另外一台Linux机器上面安装Samba客户端软件。

```bash
yum install -y cifs-utils
```

在Linux客户端，按照Samba服务的用户名、密码、共享域的顺序将相关信息写入到一个认证文件中。
为了保证不被其他人随意看到，最后把这个认证文件的权限修改为仅root管理员才能够读写：

```bash
[root@host1 ~]# vim auth.smb
username=samba
password=******
domain=SAMBA
[root@host1 ~]# chmod -Rf 600 auth.smb
```

然后在Linux客户端上创建一个用于挂载Samba服务共享资源的目录，
并把挂载信息写入到`/etc/fstab`文件中，以确保共享挂载信息在服务器重启后依然生效：

```
#
# /etc/fstab
# Created by anaconda on Sun Jan 27 21:25:27 2019
#
# Accessible filesystems, by reference, are maintained under '/dev/disk'
# See man pages fstab(5), findfs(8), mount(8) and/or blkid(8) for more info
#
/dev/mapper/centos-root /                       xfs     defaults        0 0
UUID=3acaf9d0-4cd0-40d0-af57-c84cd23fec0c /boot                   xfs     defaults        0 0
/dev/mapper/centos-swap swap                    swap    defaults        0 0
//192.168.1.20/rootdir /database cifs credentials=/root/auth.smb 0 0
```

然后执行下面的命令挂载上去：

```bash
[root@host1 ~]# mount -a
```

就可以进入目录`/database`下面看到共享文件夹下面的文件了。也可以在里面执行查看、写入、更名、删除文件等操作。
 
