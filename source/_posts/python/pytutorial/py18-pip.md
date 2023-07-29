---
title: 每天5分钟玩转Python（18） - 包管理工具pip
date: '2019-06-18 14:22:13 +0800'
toc: true
categories: [ python ]
tags: [ python教程 ]
---

安装完python后就已经安装了所有的内置模块，如果只是需要内置模块之间导入即可，
而当我们需要第三方模块的时候就需要自己去安装了。python里面最常用的是使用pip在线安装第三方模块。

其实，pip就是Python标准库中的一个包，这个包比较特殊，用它可以来管理Python标准库中其他的包。
pip支持从PyPI(<https://pypi.org/>)、版本控制以及直接从分发文件进行安装。
pip是一个命令行程序。安装pip后，会向系统添加一个pip命令，该命令可以从命令提示符运行。
目前，pip 是The Python Packaging Authority (PyPA) 推荐的 Python 包管理工具！
<!-- more -->

## 安装pip

安装pip有三种方式，我都介绍一遍。

第一种通过get-pip.py脚本安装：

```bash
curl "https://bootstrap.pypa.io/get-pip.py" -o "get-pip.py"
python get-pip.py
```

第二种先下载安装setuptoos，使用它里面的easy_install命令安装。

下载地址：<https://pypi.org/project/setuptools/#files>

```bash
tar -zxvf /root/install_venv/setuptools-41.6.0.zip -d /root/setuptools/
cd /root/setuptools && python setup.py install
easy_install pip
```

第三种方式是通过Linux的软件包安装，这里通过CentOS 7为例。
需要开启EPEL repository，然后通过yum安装pip

```bash
subscription-manager repos --enable rhel-7-server-optional-rpms --enable rhel-7-server-extras-rpms
sudo yum install python-pip
```

## 修改pip源

pip默认安装包使用的源是<<https://pypi.org/>，在国内访问可能比较慢，安装一些大的包很耗时。
目前国内已经有了很多游戏的pip源，这里介绍怎么切换至国内源。

如果机器是linux，则修改文件`~/.pip/pip.conf` (没有就创建一个)，内容如下：

```
[global]
index-url = https://pypi.tuna.tsinghua.edu.cn/simple
```

如果是windows环境，则在user目录中创建一个pip目录，如`C:\Users\xiongneng\pip`，
然后再在这个目录中新建文件pip.ini，内容如下：

```
[global]
index-url = https://pypi.tuna.tsinghua.edu.cn/simple
```

## 使用pip

安装完pip后就可以通过简单的命令安装其他python模块了，比如要安装网络爬虫scrapy：

```bash
pip install scrapy
```

### 安装指定版本

```bash
pip install scrapy==1.0.2
```

或者安装所有在包定义文件`requirements.txt`中定义的包：

```bash
pip uninstall -r requirements.txt
```

### 升级包

```bash
sudo pip install -U scrapy
```

### 卸载包

```bash
sudo pip uninstall scrapy
```

### 查看已安装包

```bash
pip list -o #列表显示，并且可查看可升级的版本
```

显示示例结果如下

```
Package    Version Latest Type
---------- ------- ------ -----
pip        19.2.3  19.3.1 wheel
setuptools 41.2.0  41.6.0 wheel
```

### 查找远程仓库可用包

```bash
pip search scrapy
```

### 下载指定的包到指定文件夹

```bash
pip download -d /data/packages/ scrapy
```

注意这里并不会安装这个包，而只是将包下载下来。

另外还可以通过包定义文件下载包：

```bash
pip download -d "/data/packages/" -r requirements.txt
```

### 安装指定的离线包

```bash
pip install --no-index --find-links=/data/packages/ scrapy
```

还可以从本地文件夹安装包定义文件中的包：

```bash
pip install --no-index --find-links=/data/packages -r requirements.txt
```

