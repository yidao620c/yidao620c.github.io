---
title: Linux用户身份和文件权限
date: '2018-02-02 15:41:32 +0800'
toc: true
categories: [ linux ]
tags: [ linux ]
---

Linux是一个多用户、多任务的操作系统，本章详细讲解如果增删改用户账号，以及文件所有者、
所属组以及其他人对文件/目录的读（r）、写（w）、执行（x）的操作权限。另外我们还可以使用SUID、
SGID与SBIT特殊权限更加灵活地设置系统权限功能，来弥补对文件设置一般权限时所带来的不足。
而文件的访问控制表（Access Control List，ACL）则还可以进一步让单一用户、
用户组对单一文件或目录进行特殊的权限设置，让文件具有能满足工作需求的最小权限。
<!--more-->

## 用户管理

在RHEL 7系统中，管理员UID为0，系统用户UID为1~999（用来运行系统服务的用户），普通用户UID从1000开始。

### useradd命令

useradd命令用来创建新的用户，格式为 `useradd [选项] 用户名`

 参数 | 说明                            
----|-------------------------------
 -d | 指定用户的家目录，默认为/home/username    
 -e | 账户的到期时间，格式为YYYY-MM-DD         
 -u | 指定用户的UID                      
 -g | 指定用户的基本组，必须已存在这个组             
 -G | 指定一个或多个扩展用户组                  
 -N | 不创建与用户同名的基本用户组                
 -s | 指定该用户的默认Shell解释器，默认为/bin/bash 

```bash
useradd -d /home/xiongneng -u 1234 -s /sbin/nologin xiongneng
```

### groupadd命令

groupadd命令用于创建用户组，格式为`groupadd [选项] 用户组名`。

```bash
groupadd xiongneng
```

### usermod命令

usermod命令用于修改用户的属性，格式为`usermod [选项] 用户名`。

 参数    | 说明                            
-------|-------------------------------
 -c    | 用户账号的备注信息                     
 -d -m | 连起来用可重新指定用户家目录并自动把旧数据转移过去     
 -e    | 账户的到期时间，格式为YYYY-MM-DD         
 -u    | 修改用户的UID                      
 -g    | 修改用户的基本组，必须已存在这个组             
 -G    | 修改用户的扩展用户组                    
 -L    | 锁定用户禁止其登录系统                   
 -U    | 解锁用户允许其登录系统                   
 -s    | 修改该用户的默认Shell解释器，默认为/bin/bash 

```bash
usermod -u 4321 -G root xiongneng
```

### passwd命令

passwd命令用于修改用户密码、过期时间、认证信息等。格式为`passwd [选项] [用户名]`。
普通用户只能修改自己的密码，并且还需要输入旧密码。而管理员root可以修改其他用户的密码，并且不需要输入旧密码。

 参数     | 说明                         
--------|----------------------------
 -l     | 锁定用户密码禁止其登录系统              
 -u     | 解锁用户密码允许其登录系统              
 -stdin | 允许通过标准输入修改用户密码             
 -d     | 使该用户可以使用空密码登录系统            
 -g     | 修改用户的基本组，必须已存在这个组          
 -e     | 强制用户下次登录时修改密码              
 -S     | 显示用户的密码是否被锁定，以及密码采用的加密算法名称 

```bash
passwd -l xiongneng
```

### userdel命令

userdel命令用户删除用户，格式为`userdel [选项] 用户名`。

 参数 | 说明         
----|------------
 -f | 强制删除用户     
 -r | 同时删除用户的家目录 

```bash
userdel -r xiongneng
```

## 文件权限管理

### 一般权限

Linux系统中一切皆文件，使用不同的字符来区分不同的文件类型。

* -: 普通文件
* d: 目录文件
* l: 链接文件
* b: 块设备文件
* c: 字符设备文件
* p: 管道文件

每个文件都有所属的所有者和所属组，并且设置所有者、所属组以及其他人对文件的读（r）、写（w）、执行（x）权限。
文件的读、写、执行权限可以简写为rwx，也可以用数字4、2、1来表述。

chmod命令可修改文件的权限属性。比如查看如下的文件权限，所属组没有任何权限。

```bash
[root@master ~]# ll anaconda-ks.cfg
-rw-------. 1 root root 1391 1月  27 2019 anaconda-ks.cfg
```

然后修改该文件的权限，将所属组权限加上读写。

```bash
[root@master ~]# chmod g+rw anaconda-ks.cfg
[root@master ~]# ll anaconda-ks.cfg 
-rw-rw----. 1 root root 1391 1月  27 2019 anaconda-ks.cfg
```

### 特殊权限

在复杂的生产环境中，单纯设置rwx权限无法满足安全和灵活性需求，便有了SUID、SGID与SBIT的特殊权限位。
这是对文件权限进行设置的特殊功能，可以与一般权限同时使用。

SUID是一种对二进制程序进行设置的特殊权限，可以让二进制程序执行者临时拥有属主的权限。
仅适用于拥有可执行权限的二进制程序有效。

注意，如果在浏览文件时，发现用户权限第三位是一个大写的 S 则表明 SUID 权限无效，
比如给一个不可执行的二进制文件设置 SGID 属性。

SGID 与 SUID类似，使用小写字母 s 表示，出现在用户组可执行权限位，
具有 SETGID 权限的文件会在其执行时，使调用者的有效用户组临时变为该文件属主的用户组，
用于临时提升权限，使调用者暂时获得该文件所属用户组的权限。

* 当 SGID 作用于可执行文件时，在执行该文件时，用户将获得该文件所属组的权限。
* 当 SGID 作用于目录时，当用户对某一目录有写和执行权限时，该用户可以在该目录下建立文件，
  如果该目具有 SGID 权限，则该用户在该目录下建立的文件都属于这个目录所属的组。

注意，如果在浏览文件时，发现用户组权限第三位是一个大写的 S 则表明 SGID 权限无效，
比如给一个不可执行的文件设置 SGID 属性。

SBIT（Sticky Bit）称为粘滞位，它出现在其他用户权限的执行位上，只能用来修饰一个目录，用于限制文件的删除。
当某一个目录拥有 SBIT 权限时，则任何一个能够在这个目录下建立文件的用户，该用户在这个目录下所建立的文件，
只有该用户自己和 root 可以删除，其他用户均不可以。

可以通过数字方式来设置这三个特殊权限。三个权限对应的数字为：SUID:4，SGID:2，SBIT:1。

```bash
chmod 4755 filename
```

此外，也可以通过符号法来设置三个特殊权限，其中 SUID 为 u+s，SGID 为 g+s，SBIT 则是 o+t。

### 文件隐藏属性

Linux系统中的文件除了具备一般权限和特殊权限之外，还有一种隐藏权限。

chattr命令用于设置文件的隐藏权限，格式为`chattr [参数] 文件名`。如果想把某个隐藏功能添加到文件上，
则需要在命令后面追加`+参数`，如果想把某个隐藏功能移出文件，则需要追加`-参数`。

 参数 | 说明                                
----|-----------------------------------
 i  | 无法对文件进行修改。如果是目录则无法新建、删除、重命名目录中的文件 
 a  | 仅允许追加内容，无法覆盖、删除内容，也无法删除文件本身。      
 S  | 文件内容变更后立即同步到硬盘                    
 s  | 彻底从硬盘中删除，不可恢复（用0填充原文件所在硬盘区域）      
 A  | 不再修改这个文件或目录的最后访问时间                
 b  | 不再修改文件或目录的存取时间                    
 u  | 删除该文件后依然保留其在硬盘中的数据，方便日后恢复         

```bash
chattr +a test.txt
```

lsattr命令用于显示文件的隐藏权限，格式为`lsattr [参数] 文件`。

```bash
[root@master ~]# lsattr test.txt 
-----a---------- test.txt
```

### 文件访问控制列表ACL

如果希望文件或目录对某个指定的用户进行单独权限控制，则需要用到ACL了。

setfacl命令用于管理文件的ACL规则，格式为`setfacl [参数] 文件名`。
针对目录文件需要使用-R递归参数，针对普通文件则使用-m参数，如果想删除ACL，则使用-b参数。

```bash
[root@master ~]# setfacl -Rm u:xiongneng:rwx /root
[root@master ~]# su - xiongneng
[xiongneng@master ~]$ cd /root
[xiongneng@master root]$ ls -l
总用量 1036
-rw-rwx---+ 1 root root    1391 1月  27 2019 anaconda-ks.cfg
-rw-rwxr--+ 1 root root       5 8月  22 16:39 test.txt
[xiongneng@master root]$ cat anaconda-ks.cfg 
```

getfacl命令用于显示文件上设置的ACL信息，格式为`getfacl 文件名`。

```bash
[xiongneng@master root]$ getfacl /root
getfacl: Removing leading '/' from absolute path names
# file: root
# owner: root
# group: root
user::r-x
user:xiongneng:rwx
group::r-x
mask::rwx
other::---
```
