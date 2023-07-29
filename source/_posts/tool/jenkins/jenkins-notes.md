---
title: Jenkins控制台日志出现中文乱码解决方法
date: 2017-04-05 23:12:33 +0800
toc: true
categories: [ 开发工具 ]
tags: [ jenkins ]
---

Jenkins放在Windows7服务器上面，通过war包启动，任务中执行python脚本时候控制台日志出现乱码。。
<!-- more -->

解决方法如下：

## 第一步：新增2个Jenkins环境变量

系统管理->系统设置

```bash
LANG=en_US.UTF-8
PYTHONIOENCODING=UTF8
```

## 第二步：新增jenkins启动参数

启动命令如下：

```bash
java -Dfile.encoding=UTF-8 -Dsun.jnu.encoding=UTF-8 -jar jenkins.war
```

**备注**

python脚本源文件也要保存为utf-8的形式，并且文件开头增加编码注释如下：

```python
# coding: utf-8
````

