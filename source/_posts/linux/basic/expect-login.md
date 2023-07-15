---
layout: post
title: 使用expect实现远程登录
date: 2021-01-27 12:21:33 +0800
toc: true
categories: [ linux ]
tags: [ expect,linux ]
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