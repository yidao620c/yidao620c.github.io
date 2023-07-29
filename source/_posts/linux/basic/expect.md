---
title: 使用expect实现自动交互脚本
date: 2021-01-27 12:21:33 +0800
toc: true
categories: [ linux ]
tags: [ linux ]
---

Expect 是一个可以通过脚本和其它交互程序通信的程序，也可以直接被用于C或C++。

可以实现什么？

* 让计算机自动回应，比如你可以登录，无需手动输入
* 开始一个游戏，如果最佳配置没有出现，一直重启直到他出现，然后把控制权交给你。
* 基于预先确定的标准，运行回话，回答问题“是”、“否”，然后把控制权交给你
* 传递环境变量，当前路径或者任何类型的信息通过rlogin, telnet, tip, su, chgrp等

安装：`yum install expect`

本篇通过一个自动远程登录脚本来演示expect的使用。
<!-- more -->

## expect参数

* -c:执行脚本前先执行的命令，可多次使用。
* -d:debug模式，可以在运行时输出一些诊断信息，与在脚本开始处使用exp_internal 1相似。
* -D:启用交换调式器,可设一整数参数。
* -f:从文件读取命令，仅用于使用#!时。如果文件名为"-"，则从stdin读取(使用"./-"从文件名为-的文件读取)。
* -i:交互式输入命令，使用"exit"或"EOF"退出输入状态。
* --:标示选项结束(如果你需要传递与expect选项相似的参数给脚本时)，可放到#!行:#!/usr/bin/expect --。
* -v:显示expect版本信息。

## 常用命令

* close:关闭当前进程的连接。
* debug:控制调试器。
* disconnect:断开进程连接(进程仍在后台运行)。
* exit:退出expect。
* exp_continue [-continue_timer]:继续执行下面的匹配。
* exp_internal [-f file] value:

```bash
# 命令行参数 
# $argv，参数数组，使用[lindex $argv n]获取，$argv 0为脚本名字
# $argc，参数个数
set username [lindex $argv 1]  # 获取第1个参数
set passwd [lindex $argv 2]    # 获取第2个参数

set timeout 30 # 设置超时

# spawn是expect内部命令，开启ssh连接
spawn ssh -l username 192.168.1.1

# 判断上次输出结果里是否包含“password:”的字符串，如果有则立即返回，否则就等待一段时间(timeout)后返回
expect "password:"

# 发送内容ispass(密码、命令等)
send "ispass\r"

# 发送内容给用户
send_user "$argv0 [lrange $argv 0 2]\n"
send_user "It's OK\r"
# 执行完成后保持交互状态，控制权交给控制台(手工操作)。否则会完成后会退出。
interact
```

## 自动telnet会话
```bash
#!/usr/bin/expect -f
set ip [lindex $argv 0 ]         # 接收第1个参数,作为IP
set userid [lindex $argv 1 ]     # 接收第2个参数,作为userid
set mypassword [lindex $argv 2 ] # 接收第3个参数,作为密码
set mycommand [lindex $argv 3 ]  # 接收第4个参数，作为命令
set timeout 10                   # 设置超时时间

# 向远程服务器请求打开一个telnet会话，并等待服务器询问用户名
spawn telnet $ip
    expect "username:"
    # 输入用户名，并等待服务器询问密码
    send "$userid\r"
    expect "password:"
    # 输入密码，并等待键入需要运行的命令
    send "$mypassword\r"
    expect "%"
    # 输入预先定好的密码，等待运行结果
    send "$mycommand\r"
    expect "%"
    # 将运行结果存入到变量中，显示出来或者写到磁盘中
    set results $expect_out(buffer)
    # 退出telnet会话，等待服务器的退出提示EOF
    send "exit\r"
    expect eof
```

## 自动建立FTP会话
```bash
#!/usr/bin/expect -f
set ip [lindex $argv 0 ]         # 接收第1个参数,作为IP
set userid [lindex $argv 1 ]     # 接收第2个参数,作为Userid
set mypassword [lindex $argv 2 ] # 接收第3个参数,作为密码
set timeout 10                   # 设置超时时间

# 向远程服务器请求打开一个FTP会话，并等待服务器询问用户名
spawn ftp $ip
    expect "username:"
    # 输入用户名，并等待服务器询问密码
    send "$userid\r"
    expect "password:"
    # 输入密码，并等待FTP提示符的出现
    send "$mypassword\r"
    expect "ftp>"
    # 切换到二进制模式，并等待FTP提示符的出现
    send "bin\r"
    expect "ftp>"
    # 关闭ftp的提示符
    send "prompt\r"
    expect "ftp>"
    # 下载所有文件
    send "mget *\r"
    expect "ftp>"
    # 退出此次ftp会话，并等待服务器的退出提示EOF
    send "bye\r"
    expect eof
```

## 自动登录ssh执行命令
```bash
#!/usr/bin/expect
set IP     [lindex $argv 0]
set USER   [lindex $argv 1]
set PASSWD [lindex $argv 2]
set CMD    [lindex $argv 3]

spawn ssh $USER@$IP $CMD
expect {
    "(yes/no)?" {
        send "yes\r"
        expect "password:"
        send "$PASSWD\r"
        }
    "password:" {send "$PASSWD\r"}
    "* to host" {exit 1}
    }
expect eof
```

## 自动登录ssh
```bash
#!/usr/bin/expect -f  
set ip [lindex $argv 0 ]         # 接收第1个参数,作为IP
set username [lindex $argv 1 ]   # 接收第2个参数,作为username
set mypassword [lindex $argv 2 ] # 接收第3个参数,作为密码
set timeout 10                   # 设置超时时间 

spawn ssh $username@$ip       # 发送ssh请求
expect {                      # 返回信息匹配 
"*yes/no" { send "yes\r"; exp_continue}  # 第一次ssh连接会提示yes/no,继续  
"*password:" { send "$mypassword\r" }    # 出现密码提示,发送密码  
} 
interact        # 交互模式,用户会停留在远程服务器上面
```

## 批量登录ssh服务器执行操作范例，for语法
```bash
#!/usr/bin/expect
for {set i 10} {$i <= 12} {incr i} {
    set timeout 30
    set ssh_user [lindex $argv 0]
    spawn ssh -i .ssh/$ssh_user abc$i.com

    expect_before "no)?" {
    send "yes\r" }
    sleep 1
    expect "password*"
    send "hello\r"
    expect "*#"
    send "echo hello expect! > /tmp/expect.txt\r"
    expect "*#"
    send "echo\r"
}
exit
```

## 批量登录ssh并执行命令，foreach语法
```bash
#!/usr/bin/expect
if {$argc!=2} {
    send_user "usage: ./expect ssh_user password\n"
    exit
}
foreach i {11 12} {
    set timeout 30
    set ssh_user [lindex $argv 0]
    set password [lindex $argv 1]
    spawn ssh -i .ssh/$ssh_user root@xxx.yy.com
    expect_before "no)?" {
    send "yes\r" }
    sleep 1

    expect "Enter passphrase for key*"
    send "password\r"
    expect "*#"
    send "echo hello expect! > /tmp/expect.txt\r"
    expect "*#"
    send "echo\r"
}
exit
```

## 从命令行获取服务器IP，foreach语法，expect嵌套
```bash
#!/usr/bin/expect
# 使用方法: script_name ip1 ip2 ip3 ...

set timeout 20
if {$argc < 1} {
  puts "Usage: script IPs"
  exit 1
}
# 替换你自己的用户名
set user "username"
#替换你自己的登录密码
set password "yourpassword"

foreach IP $argv {
spawn ssh $user@$IP

expect \
  "(yes/no)?" {
    send "yes\r"
    expect "password:?" {
      send "$password\r"
    }
  } "password:?" {
    send "$password\r"
}

expect "\$?"
# 替换你要执行的命令
send "last\r"
expect "\$?"
sleep 10
send "exit\r"
expect eof
}
```

## 批量ssh执行命令，用shell调用tclsh方式、多进程同时执行
```bash
#!/bin/sh
# -*- tcl -*- \
exec tclsh $0 "$@"
package require Expect
set username [lindex $argv 0]
set password [lindex $argv 1]
set argv [lrange $argv 2 end]
set prompt "(%|#|\\$) $"
foreach ip $argv {
    spawn ssh -t $username@$ip sh
    lappend ids $spawn_id
}
expect_before -i ids eof {
    set index [lsearch $ids $expect_out(spawn_id)]
    set ids [lreplace $ids $index $index]
    if [llength $ids] exp_continue
}
expect -i ids "(yes/no)\\?" {
    send -i $expect_out(spawn_id) yes\r
    exp_continue
} -i ids "Enter passphrase for key" {
    send -i $expect_out(spawn_id) \r
    exp_continue
} -i ids "assword:" {
    send -i $expect_out(spawn_id) $password\r
    exp_continue
} -i ids -re $prompt {
    set spawn_id $expect_out(spawn_id)
    send "echo hello; exit\r"
    exp_continue
} timeout {
    exit 1
}
```

## ssh登录过程常规提示文字
```
The authenticity of host '192.168.17.35 (192.168.17.35)' can't be established.
RSA key fingerprint is 25:e8:4c:89:a3:b2:06:ee:de:66:c7:7e:1b:fa:1c:c5.
Are you sure you want to continue connecting (yes/no)?

Warning: Permanently added '192.168.17.35' (RSA) to the list of known hosts.
Enter passphrase for key '/data/key/my_dsa':

Last login: Sun Jan 26 13:39:37 2014 from 192.168.11.143
[root@master003 ~]#
root@192.168.16.90's password:

Last login: Thu Jan 23 17:50:43 2014 from 192.168.11.102
[root@lvsmaster ~]#
```

## 批量主机登录例子
```bash
#!/bin/bash

#########################################################################
# 演示通过expect实现批量主机操作
# 任务：以普通用户SSH登录远程主机后su切换至root再执行命令，自动完成口令输入。
# author: xiongneng
# 使用方法：
# 首先准备一个配置文件config.txt，把主机IP、普通用户密码、root密码。每行一条，逗号隔开。
# 然后执行 ./xx.sh config.txt
#########################################################################

main() {
    config_file=$1
    user_name=user
    for line in $(cat $config_file); do
        IFS=',' read -r a record <<<"$line"
        ip=$(record[0])
        user_pwd=$(record[1])
        root_pwd=$(record[2])
        /usr/bin/expect <<-EOF
        set timeout 10
        log_file error.log
        spawn -noecho ssh -tq $user_name@$ip "su -c \"
        chown root:root /etc/ssh/*.key && chmod 400 /etc/ssh/*.key;
        chown root:root /etc/ssh/*.key.pub && chmod 400 /etc/ssh/*.key.pub;
        systemctl restart sshd\""
        expect {
            "*yes/no" {send "yes\r"; exp_continue}
            "*s password" {send "${user_pwd}\r"}
            "*No route to host" {send_log "\nno route to ip, =====$ip=====\n"; exit}
            timeout {send_log "\nconnect ip timeout, =====$ip=====\n"; exit}
        }
        expect {
            "*s password" {send "\nuser password error, =====$ip=====\n"; exit}
            "Password:" {send "${root_pwd}\r"; exp_continue}
            "*Permission denied" {send_log "\nroot password error, =====$ip=====\n"; exit}
        }
        
EOF
    echo "ip=$ip process finished"
    done
    echo "All tasks finish successfully."
}

if [[ "$#" < 1 ]]; then
    echo "[error] input invalid, usage: "
    echo "./xx.sh config.txt"
    exit 1
fi

main $1
```
