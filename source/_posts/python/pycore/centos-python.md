---
layout: post
title: centos6.x安装python2.7
date: 2015-10-10 19:02:42 +0800
toc: true
categories: [ python ]
tags: [ python ]
---

更新系统和开发工具集，下面所有的指令都在root用户下完成

```bash
yum -y update
yum groupinstall -y 'development tools'
```

另外还需要安装 python 工具需要的额外软件包 SSL, bz2, zlib

```bash
yum install -y zlib-devel bzip2-devel openssl-devel xz-libs wget
```

<!-- more -->

## 源码安装Python 2.7.x

```bash
wget http://www.python.org/ftp/python/2.7.11/Python-2.7.11.tar.xz
xz -d Python-2.7.11.tar.xz
tar -xvf Python-2.7.11.tar
```

## 详细步骤

```bash
# 进入目录:
cd Python-2.7.11
# 运行配置 configure:
./configure --prefix=/usr/local
# 编译安装:
make
make altinstall
# 检查 Python 版本:
[root@dbmasterxxx ~]# python2.7 -V
Python 2.7.11
```

## 设置PATH

设置系统变量并建立软连接将新版本的Python，编辑/etc/profile：

```bash
PATH=/usr/local/bin:$PATH
```

保存后，再执行：

```bash
unlink /usr/bin/python
ln -s /usr/local/bin/python2.7  /usr/bin/python

## 检查
[root@dbmasterxxx ~]# python -V
Python 2.7.11
[root@dbmasterxxx ~]# which python
/usr/bin/python

```

## 安装 setuptools

```bash
#获取软件包，如果下面的wget获取不到就直接去pypi网站下载
wget --no-check-certificate https://pypi.python.org/packages/source/s/setuptools/setuptools-20.3.tar.gz
# 解压:
tar -xvf setuptools-20.3.tar.gz
cd setuptools-20.3
# 使用 Python 2.7 安装 setuptools
python2.7 setup.py install
```

## 安装 PIP

```bash
curl https://bootstrap.pypa.io/get-pip.py | python2.7 -
```

## 修复 yum 工具

```bash
[root@dbmasterxxx ~]# which yum
/usr/bin/yum
#修改 yum中的python
将第一行 #!/usr/bin/python 改为 #!/usr/bin/python2.6
```

此时yum就ok啦！

## 总结

Python版本升级过很多遍，每次都有问题，此方法来自互联网，经过使用，没有问题，特此总结一下
