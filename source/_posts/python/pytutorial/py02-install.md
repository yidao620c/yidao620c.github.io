---
title: 每天5分钟玩转Python（02） - 安装Python
date: '2019-06-02 09:35:12 +0800'
toc: true
categories: [ python ]
tags: [ python教程 ]
---

要学习和运行Python得先安装才行，安装后会得到Python解释器、命令行交互环境和一个简单的集成开发环境。

由于历史原因，目前Python有两个版本：一个是2.x版，一个是3.x版。2.x版本很快就不会被支持了，
现在所有python编写的软件都会升级到3.x，所以这个教程就直接以最新的3.8版本为基础来讲解。
同时建议所有初学者直接学习python 3.x版本。
<!-- more -->

## 下载和安装

Python是跨平台的，所以不管是windows、linux和mac都能安装。

### 在Mac上安装Python

Mac系统自带python版本一般是2.7，如果需要安装最新的3.8版本，有两个办法：

1. 从Python官网下载Python 3.8的[安装程序](https://www.python.org/ftp/python/3.8.0/python-3.8.0-macosx10.9.pkg)
   ，下载后双击运行并安装；
2. 如果安装了Homebrew，直接通过命令`brew install python3`安装即可。

### 在Linux上安装Python

一般的Linux发行版也会内置Python，centos7内置的是2.7。可以先执行`python -V`看看版本号。
如果想要升级到最新的3.8版本，推荐一行命令搞定（仅适用通过apt或yum来安装包的系统\[ubuntu/red hat/centos]）

```bash
curl https://bc.gongxinke.cn/downloads/install-python-latest | bash
```

### 在Windows上安全Python

现在的windows系统一般都是64位系统了，
所以建议你去下载[64位安装版本](https://www.python.org/ftp/python/3.8.0/python-3.8.0-amd64.exe)

下载下来后，双击安装包，选中"Add Python 3.8 to PATH" 这个选项，点击"Install Now"开始安装。

![](https://xnstatic-1253397658.file.myqcloud.com/p02_install.png)

## 运行Python

安装成功后，打开一个命令行窗口，输入python，会进入交互式环境，如下图

![](https://xnstatic-1253397658.file.myqcloud.com/p02_cmd.png)

好了，现在你已经成功将python安装到机子上了。太棒了，离成功只差2万公里了。
