---
title: 一些有趣的python技巧
date: 2016-05-20 19:32:56 +0800
toc: true
categories: [ python ]
tags: [ python ]
---

python有时候简单起来连我自己都怕，有时候其他语言需要几十写出来的python几行搞定。
这里经常收集一些有趣的东西还是很好玩的。
<!-- more -->

## 简单的HTTP服务器

你想快速简单的分享目录下的文件吗？可以这样做：

```
cd $HOME/work/

# Python2
python -m SimpleHTTPServer

# Python 3
python3 -m http.server 8000
```

然后别人就可以打开`http://ip:8000/`来访问这个简单的Web服务器了，
如果该文件夹里面有个index.html就显示它，如果没有就显示文件和目录列表。
这样就能给别人快速分享文件或展示你的网站。

## 简单的FTP服务器

```
pip install pyftpdlib
python -m pyftpdlib -p 21
```

可选参数

* -i 指定IP地址（默认为本机的IP地址）
* -p 指定端口（默认为2121）
* -w 写权限（默认为只读）
* -d 指定目录 （默认为当前目录）
* -u 指定用户名登录
* -P 设置登录密码

## 优雅地打印

下面的方式可以用优雅的方式打印字典和列表:

```
from pprint import pprint
pprint(my_dict)
```

这用于字典打印是非常高效的，如果你想从文件中快速优雅的打印出json，可以这样做：

```
cat file.json | python -m json.tool
```
