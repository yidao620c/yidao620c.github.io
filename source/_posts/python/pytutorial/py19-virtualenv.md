---
title: 每天5分钟玩转Python（19） - 安装虚拟环境
date: '2019-06-19 15:22:56 +0800'
toc: true
categories: [ python ]
tags: [ python教程 ]
---

如果正常使用pip安装会将软件包安装到python系统包目录下面，也就是site-packages目录。
通常我们需要对环境进行隔离以防止别的包影响到。这时候需要安装虚拟环境virtualenv了。
<!-- more -->

## 安装virtualenv

virtualenv可用于创建独立的 Python 环境，它会创建一个包含项目所必须要的执行文件。

```bash
sudo pip install -U virtualenv
```

## 创建虚拟环境

如下命令表示在当前目录下创建一个名叫 `myenv` 的目录（虚拟环境），
该目录下包含了独立的 Python 运行程序，以及虚拟环境运行pip命令安装的包。

```bash
virtualenv myenv
```

默认情况下，虚拟环境会依赖系统环境中已经安装过的包，
就是说系统中已经安装好的第三方包也会安装在虚拟环境中，如果不想依赖这些包，
那么可以加上参数 `--no-site-packages` 建立虚拟环境

```bash
# 不继承系统的site-packages中的模块
virtualenv --no-site-packages myenv
```

## 激活虚拟环境

```bash
source myenv/bin/activate
```

## 退出虚拟环境

```bash
deactivate
```

## 删除虚拟环境

退出后直接删除myenv文件夹即可

## pip安装第三方包

```bash
(未激活) pip install -E myenv scrapy
(激活)   pip install scrapy
```

## 整个虚拟环境导出

有时候你想将自己创建的虚拟环境共享给其他人用，可以先将其导出为一个文本文件，命令如下：

```bash
pip freeze -E django-mimo > requirements.txt
```

## 整个虚拟环境导入

使用上面导出的虚拟环境命令如下：

```bash
pip install -E django-mimo -r requirements.txt
```
