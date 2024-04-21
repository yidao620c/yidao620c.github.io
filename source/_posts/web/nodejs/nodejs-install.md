---
title: Nodejs的安装和配置
date: 2024-03-17 09:08:03 +0800
toc: true
categories: [ web ]
tags: [ web, nodejs ]
---

Node.js平台是在后端运行JavaScript代码，所以，必须首先在本机安装Node环境。

Node.js官网地址：https://nodejs.org

如果是windows系统，可以直接下载安装文件安装。

CentOS系统安装指定的版本方法：
```
curl --silent --location https://rpm.nodesource.com/setup_18.x | sudo bash
yum -y install nodejs
```

Ubuntu系统安装指定的版本：
```
sudo apt-get update
curl -sL https://deb.nodesource.com/setup_18.x | sudo -E bash -
sudo apt-get install -y nodejs
```

<!--more-->

npm若报错 （1）关闭命令窗口再打开 （2）不可行时再执行以下命令后，重新启动bash
```
sudo apt install note
sudo apt-get install -y build-essential
```
最新的node.js已经集成了npm，不过需要经常更新：

```bash
npm install npm@latest -g
npm --version
```

查看npm源
```
sudo npm config get registry
```

安装淘宝源和cnpm
```bash
npm cache clean --force
npm config set registry https://registry.npmmirror.com
npm install -g cnpm --registry=https://registry.npmmirror.com
```

之后使用cnpm命令代替npm命令即可。
