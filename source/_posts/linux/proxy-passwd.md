---
layout: post
title: 设置代理时候保护个人密码
date: '2018-09-28 14:04:10 +0800'
toc: true
categories: linux
tags:
  - proxy
abbrlink: 12127
---

一般设置代理方式是，全局的代理设置`vi /etc/profile`

添加下面内容
```bash
export http_proxy = http://username:password@yourproxy.com:8080/
export ftp_proxy = http://username:password@yourproxy:8080/
```

但是这种直接在配置文件里面写自己域账号的明文密码很不安全，如果是几个人共享一台机器，其他人可以直接看到你的密码。
<!-- more -->

解决办法是将自己的域密码进行加密存储，步骤如下：

先创建一个明文密码文件`/etc/profile.d/pw.txt`，内容就是域密码，然后执行：

```bash
openssl enc -aes-256-cbc -in /etc/profile.d/pw.txt -out /etc/profile.d/pw.bin
```
得到加密后的密文，然后删除明文密码文件pw.txt。

接下来在`/etc/profile`中创建2个别名命令：

1. 设置代理的别名
```bash
alias myproxy='PW=`openssl aes-256-cbc -d -in /etc/profile.d/pw.bin`; PROXY="http://$USER:$PW@yourproxy.com:8080"; export http_proxy=$PROXY; export https_proxy=$PROXY; export ftp_proxy=$PROXY'
```

2. 清空代理的别名
```bash
alias clearproxy ='export http_proxy=; export https_proxy=; ftp_proxy='
```

注：在shell会话期间，密码在用户环境中可用（并且可读）。如果您想在使用后从环境中清除它，可以使用clearproxy这个别名

使用的时候，先执行myproxy，会提示你输入加密域密码时候的密码。然后开始使用，使用完毕可以执行clearproxy清空

