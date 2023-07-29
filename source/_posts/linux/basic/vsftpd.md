---
title: CentOS7搭建vsftpd服务
date: '2018-02-08 22:14:22 +0800'
toc: true
categories: [ linux ]
tags: [ linux ]
---

FTP是一种在互联网中进行文件传输的协议，基于客户端/服务器模式，默认使用20、21号端口，
其中端口20（数据端口）用于进行数据传输，端口21（命令端口）用于接受客户端发出的相关FTP命令与参数。
FTP服务器普遍部署于内网中，具有容易搭建、方便管理的特点。
而且有些FTP客户端工具还可以支持文件的多点下载以及断点续传技术，因此FTP服务应用相当广泛。

FTP服务器是按照FTP协议在互联网上提供文件存储和访问服务的主机，FTP客户端则是向服务器发送连接请求，
以建立数据传输链路的主机。FTP协议有下面两种工作模式。

* 主动模式：FTP服务器主动向客户端发起连接请求。
* 被动模式：FTP服务器等待客户端发起连接请求（FTP的默认工作模式）。

防火墙一般是用于过滤从外网进入内网的流量，因此有些时候需要将FTP的工作模式设置为主动模式，才可以传输数据。

vsftpd（very secure ftp daemon，非常安全的FTP守护进程）是一款运行在Linux操作系统上的FTP服务程序，
不仅完全开源而且免费，此外，还具有很高的安全性、传输速度，以及支持虚拟用户验证等其他FTP服务程序不具备的特点。
<!--more-->

## 安装服务端vsftpd

首先安装服务端程序vsftpd

```bash
yum install vsftpd
```

vsftpd服务程序的主配置文件是`/etc/vsftpd/vsftpd.conf`。

```bash
[root@master ~]# mv /etc/vsftpd/vsftpd.conf /etc/vsftpd/vsftpd.conf_bak
[root@master ~]# grep -v "#" /etc/vsftpd/vsftpd.conf_bak > /etc/vsftpd/vsftpd.conf
[root@master ~]# cat /etc/vsftpd/vsftpd.conf
anonymous_enable=YES
local_enable=YES
write_enable=YES
local_umask=022
dirmessage_enable=YES
xferlog_enable=YES
connect_from_port_20=YES
xferlog_std_format=YES
listen=NO
listen_ipv6=YES
pam_service_name=vsftpd
userlist_enable=YES
tcp_wrappers=YES
```

常用配置说明：

 参数                               | 说明                                 
----------------------------------|------------------------------------
 listen=[YES/NO]                  | 是否以独立运行的方式监听服务                     
 listen_address=IP地址              | 设置要监听的IP地址                         
 listen_port=21                   | 设置FTP服务的监听端口                       
 download_enable＝[YES/NO]         | 是否允许下载文件                           
 write_enable=[YES/NO]	           | 设置是否可写权限                           
 userlist_enable=[YES/NO]         | 开启用户作用名单文件功能                       
 userlist_deny=[YES/NO]           | 启用"禁止用户名单"，名单文件为ftpusers和user_list 
 max_clients=0                    | 最大客户端连接数，0为不限制                     
 max_per_ip=0                     | 同一IP地址的最大连接数，0为不限制                 
 anonymous_enable=[YES/NO]        | 是否允许匿名用户访问                         
 anon_upload_enable=[YES/NO]      | 是否允许匿名用户上传文件                       
 anon_umask=022                   | 匿名用户上传文件的umask值                    
 anon_root=/var/ftp               | 匿名用户的FTP根目录                        
 anon_mkdir_write_enable=[YES/NO] | 是否允许匿名用户创建目录                       
 anon_other_write_enable=[YES/NO] | 是否开放匿名用户的其他写入权限（包括重命名、删除等操作权限）     
 anon_max_rate=0                  | 匿名用户的最大传输速率（字节/秒），0为不限制            
 local_enable=[YES/NO]            | 是否允许本地用户登录FTP                      
 local_umask=022                  | 本地用户上传文件的umask值                    
 local_root=/var/ftp              | 本地用户的FTP根目录                        
 chroot_local_user=[YES/NO]       | 是否将用户权限禁锢在FTP目录，以确保安全              
 local_max_rate=0                 | 本地用户最大传输速率（字节/秒），0为不限制             

## 安装客户端ftp

再安装客户端程序ftp，ftp是Linux系统中以命令行界面的方式来管理FTP传输服务的客户端工具。
我们首先手动安装这个ftp客户端工具：

```bash
yum install ftp
```

### 目录命令

* pwd 显示远程计算机上的当前目录
* ls/dir 列出当前远程目录的内容，可以使用该命令在Linux下的任何合法的ls选项
* cd 移动到cd 后的目录
* cdup/cd .. 返回上一级目录
* lcd 列出当前本地目录路径

必须具有权限

* mkdir 在远程系统中创建目录
* rname 重命名一个文件或目录
* redir 删除远程目录
* delete 删除远程文件
* mdelete 删除多个远程文件

### 文件上传下载

* binary 用于二进制文件传送（图像文件等）
* ascii 用于文本文件传送
* get/mget 在当前远程目录下复制（一个/多个）文件到本地文件系统当前目录
* put/mput 从当前目录把文件复制到当前远程目录

## 三种认证方式

vsftpd作为更加安全的文件传输的服务程序，允许用户以三种认证模式登录到FTP服务器上。

### 匿名访问模式

是一种最不安全的认证模式，任何人都可以无需密码验证而直接登录到FTP服务器。
编辑配置文件`/etc/vsftpd/vsftpd.conf`，修改部分如下。

```
anonymous_enable=YES
anon_umask=022
anon_upload_enable=YES
anon_mkdir_write_enable=YES
anon_other_write_enable=YES
```

修改后保存并重启vsftpd服务，同时设置开启启动

```bash
systemctl restart vsftpd
systemctl enable vsftpd
```

现在就可以在客户端执行ftp命令连接到远程的FTP服务器了。在vsftpd服务程序的匿名开放认证模式下，
其账户统一为anonymous，密码为空。而且在连接到FTP服务器后，默认访问的是`/var/ftp`目录。
我们可以切换到该目录下的pub目录中，然后尝试创建一个新的目录文件，以检验是否拥有写入权限：

```bash
chown -Rf ftp /var/ftp/pub
[root@master ~]# ftp localhost
Trying ::1...
Connected to localhost (::1).
220 (vsFTPd 3.0.2)
Name (localhost:root): anonymous
331 Please specify the password.
Password:
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> cd pub
250 Directory successfully changed.
ftp> mkdir files
257 "/pub/files" created
ftp> exit
221 Goodbye.
```

### 本地用户模式

是通过Linux系统本地的账户密码信息进行认证的模式，相较于匿名开放模式更安全，而且配置起来也很简单。
但是如果被黑客破解了账户的信息，就可以畅通无阻地登录FTP服务器，从而完全控制整台服务器。

本地用户模式使用的权限参数以及作用

 参数                    | 说明                                 
-----------------------|------------------------------------
 anonymous_enable=NO   | 禁止匿名访问模式                           
 local_enable=YES      | 允许本地用户模式                           
 write_enable=YES	     | 设置可写权限                             
 local_umask=022       | 本地用户上传文件的umask值                    
 userlist_enable=YES   | 开启用户作用名单文件功能                       
 userlist_deny=YES     | 启用"禁止用户名单"，名单文件为ftpusers和user_list 
 local_root=/var/ftp   | 本地用户的FTP根目录                        
 chroot_local_user=YES | 是否将用户权限禁锢在FTP目录，以确保安全              
 local_max_rate=0      | 本地用户最大传输速率（字节/秒），0为不限制             

同样修改配置文件`/etc/vsftpd/vsftpd.conf`，将上面的参数设置好重启服务。

另外由于配置了`userlist_deny=YES`，
则在`/etc/vsftpd/user_list`和`/etc/vsftpd/ftpusers`中的用户不能登陆。

可手动修改这两个文件，将root去掉。然后再用root去登陆ftp

```bash
[root@master ~]# ftp localhost
Trying ::1...
Connected to localhost (::1).
220 (vsFTPd 3.0.2)
Name (localhost:root): root
331 Please specify the password.
Password:
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> 
```

在采用本地用户模式登录FTP服务器后，默认访问的是该用户的家目录，所以肯定有权限创建文件。

### 虚拟用户模式

是这三种模式中最安全的一种认证模式，它需要为FTP服务单独建立用户数据库文件，
虚拟出用来进行口令验证的账户信息，而这些账户信息在服务器系统中实际上是不存在的，
仅供FTP服务程序进行认证使用。这样，即使黑客破解了账户信息也无法登录服务器，从而有效降低了破坏范围和影响。

第1步：创建用于进行FTP认证的用户数据库文件，其中奇数行为账户名，偶数行为密码。
例如，我们分别创建出zhangsan和lisi两个用户，密码均为redhat：

```
cd /etc/vsftpd/
vim vuser.list
zhangsan
redhat
lisi
redhat
```

但是，明文信息既不安全，也不符合让vsftpd服务程序直接加载的格式，
因此需要使用db_load命令用哈希（hash）算法将原始的明文信息文件转换成数据库文件，
并且降低数据库文件的权限（避免其他人看到数据库文件的内容），然后再把原始的明文信息文件删除。

```bash
[root@master vsftpd]# db_load -T -t hash -f vuser.list vuser.db
[root@master vsftpd]# file vuser.db
vuser.db: Berkeley DB (Hash, version 9, native byte-order)
[root@master vsftpd]# chmod 600 vuser.db
[root@master vsftpd]# rm -f vuser.list
```

第2步：创建vsftpd服务程序用于存储文件的根目录以及虚拟用户映射的系统本地用户。
FTP服务用于存储文件的根目录指的是，当虚拟用户登录后所访问的默认位置。

由于Linux系统中的每一个文件都有所有者、所属组属性，例如使用虚拟账户"张三"新建了一个文件，
但是系统中找不到账户"张三"，就会导致这个文件的权限出现错误。
为此，需要再创建一个可以映射到虚拟用户的系统本地用户。简单来说，
就是让虚拟用户默认登录到与之有映射关系的这个系统本地用户的家目录中，
虚拟用户创建的文件的属性也都归属于这个系统本地用户，从而避免Linux系统无法处理虚拟用户所创建文件的属性权限。

为了方便管理FTP服务器上的数据，可以把这个系统本地用户的家目录设置为/var目录
（该目录用来存放经常发生改变的数据）。并且为了安全起见，我们将这个系统本地用户设置为不允许登录FTP服务器，
这不会影响虚拟用户登录，而且还可以避免黑客通过这个系统本地用户进行登录。

```bash
[root@master vsftpd]# useradd -d /var/ftproot -s /sbin/nologin virtual
[root@master vsftpd]# ls -ld /var/ftproot/
drwx------ 2 virtual virtual 62 8月  23 23:10 /var/ftproot/
[root@master vsftpd]# chmod -Rf 755 /var/ftproot/
```

第3步：建立用于支持虚拟用户的PAM文件。

PAM（可插拔认证模块）是一种认证机制，通过一些动态链接库和统一的API把系统提供的服务与认证方式分开，
使得系统管理员可以根据需求灵活调整服务程序的不同认证方式。

新建一个用于虚拟用户认证的PAM文件vsftpd.vu，
其中PAM文件内的"db="参数为使用db_load命令生成的账户密码数据库文件的路径，但不用写数据库文件的后缀：

```bash
vim /etc/pam.d/vsftpd.vu
auth       required     pam_userdb.so db=/etc/vsftpd/vuser
account    required     pam_userdb.so db=/etc/vsftpd/vuser
```

第4步：在vsftpd服务程序的主配置文件中通过pam_service_name参数将PAM认证文件的名称修改为vsftpd.vu，
PAM作为应用程序层与鉴别模块层的连接纽带，可以让应用程序根据需求灵活地在自身插入所需的鉴别功能模块。
当应用程序需要PAM认证时，则需要在应用程序中定义负责认证的PAM配置文件，实现所需的认证功能。

利用PAM文件进行认证时使用的参数以及作用

 参数                          | 说明                    
-----------------------------|-----------------------
 anonymous_enable=NO         | 禁止匿名访问模式              
 local_enable=YES            | 允许本地用户模式              
 guest_enable=YES	           | 开启虚拟用户模式              
 guest_username=virtual	     | 指定虚拟用户账户              
 pam_service_name=vsftpd.vu	 | 指定PAM文件               
 chroot_local_user=YES       | 是否将用户权限禁锢在FTP目录，以确保安全 

第5步：为虚拟用户设置不同的权限。虽然账户zhangsan和lisi都是用于vsftpd服务程序认证的虚拟账户，
但是我们依然想对这两人进行区别对待。比如，允许张三上传、创建、修改、查看、删除文件，只允许李四查看文件。
这可以通过vsftpd服务程序来实现。只需新建一个目录，在里面分别创建两个以zhangsan和lisi命名的文件，
其中在名为zhangsan的文件中写入允许的相关权限（使用匿名用户的参数）：

```bash
[root@master vsftpd]# mkdir /etc/vsftpd/vusers_dir/
[root@master vsftpd]# cd /etc/vsftpd/vusers_dir/
[root@master vusers_dir]# touch lisi
[root@master vusers_dir]# vim zhangsan
anon_upload_enable=YES
anon_mkdir_write_enable=YES
anon_other_write_enable=YES
```

然后再次修改vsftpd主配置文件，通过添加user_config_dir参数来定义这两个虚拟用户不同权限的配置文件所存放的路径。

```
user_config_dir=/etc/vsftpd/vusers_dir
```

然后重启服务后，测试用zhansan和lisi登陆创建文件夹。一个有权限一个没有：

```bash
[root@master ~]# ftp  localhost
Trying ::1...
Connected to localhost (::1).
220 (vsFTPd 3.0.2)
Name (localhost:root): zhangsan
331 Please specify the password.
Password:
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> mkdir files
257 "/files" created
ftp> exit
221 Goodbye.
[root@master ~]# ftp  localhost
Trying ::1...
Connected to localhost (::1).
220 (vsFTPd 3.0.2)
Name (localhost:root): lisi
331 Please specify the password.
Password:
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> mkdir files
550 Create directory operation failed.
ftp> 
```
