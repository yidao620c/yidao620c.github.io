---
title: 每天5分钟玩转Python（20） - 打包与发布
date: '2019-06-20 12:35:22 +0800'
toc: true
categories: [ python ]
tags: [ python教程 ]
---

当需要将写的程序打包分发出去的时候，就要使用到setuptools工具了，这里我通过一个实际例子来介绍它的使用方法。
之前我自己写过一个rpc模块叫xnrpc：

* github项目地址：<https://github.com/yidao620c/xnrpc>
* pipi模块地址：<https://pypi.python.org/pypi/xnrpc>

<!-- more -->

## 软件包归档格式

Python的软件包一开始是没有官方的标准分发格式的。比如Java有jar包或者war包作为分发格式，Python则什么都没有。
后来不同的工具都开始引入一些比较通用的归档格式。比如，setuptools引入了Egg格式。
但是，这些都不是官方支持的，存在元数据和包结构彼此不兼容的问题。因此，为了解决这个问题，
PEP 427定义了新的分发包标准，名为Wheel。目前pip和setuptools工具都支持Wheel格式。
这里我们简单总结一下常用的分发格式：

* tar.gz格式：这个就是标准压缩格式，里面包含了项目元数据和代码，可以使用Python setup.py sdist命令生成。
* egg格式：这个本质上也是一个压缩文件，只是扩展名换了，里面也包含了项目元数据以及源代码。这个格式由setuptools项目引入。
  可以通过命令`Python setup.py bdist_egg`命令生成。
* whl格式：这个是Wheel包，也是一个压缩文件，只是扩展名换了，里面也包含了项目元数据和代码，还支持免安装直接运行。
  whl分发包内的元数据和egg包是有些不同的。这个格式是由PEP 427引入的。可以通过命令`Python setup.py bdist_wheel`生成。

<http://pythonwheels.com/> 网站展示了使用Wheels发行的python模块在PyPI上的占有率，推荐使用wheel包。

## .egg-info和.dist-info目录

如果你到系统中安装Python库的路径下看看，就能看到很多名称以.egg-info或者以.dist-info结尾的目录。这些目录的内容就是这个库的元数据，
是从库的分发包中拷贝出来的。其中.egg-info类型的目录来自于Egg格式的分发包，.dist-info类型的目录来自于Wheel格式的分发包

## 项目目录结构

xnrpc项目的目录结果如下

![](https://xnstatic-1253397658.file.myqcloud.com/pysetup001.png)

项目最顶层的目录为"xnrpc"，其中与打包最相关的文件是setup.py，
这里面最核心的文件就是这个setup.py了，我们看看里面写什么：

```python
#!/usr/bin/env python
# -*- encoding: utf-8 -*-

from setuptools import setup, find_packages

setup(
    name='xnrpc',
    version='1.0.0',
    packages = find_packages(),
    # Project uses , so ensure
    install_requires=[
        "gevent>=1.1.2",
        "zerorpc>=0.6.0",
    ],
    description='simple rpc based on zerorpc and gevent',
    long_description=open("README.rst").read(),
    url='https://github.com/yidao620c/xnrpc',
    author='Xiong Neng',
    author_email='yidao620@gmail.com',
    license='Apache License 2.0',
    classifiers=[
        'Development Status :: 4 - Beta',
        'Intended Audience :: Developers',
        'Topic :: Software Development :: Build Tools',
        'License :: OSI Approved :: MIT License',
        'Programming Language :: Python :: 2.6',
        'Programming Language :: Python :: 2.7',
        'Programming Language :: Python :: 3',
        'Programming Language :: Python :: 3.3',
        'Programming Language :: Python :: 3.4',
        'Programming Language :: Python :: 3.5',
    ],
    package_data = {
        # If any package contains *.txt or *.rst files, include them:
        '': ['*.txt', '*.rst'],
        # include any *.msg files found in the 'test' package, too:
        'test': ['*.msg'],
    },
    # The data_files option can be used to specify additional files
    # needed by the module distribution: configuration files,
    # message catalogs, data files
    data_files=[('etc/xnrpc', ['etc/xnrpc.conf']), ],
    cmdclass={'install': CustomInstallCommand},
    keywords=['xnrpc', 'gevent', 'zerorpc'],
    entry_points={
        # "xnrpc.registered_commands": [
        #     "upload = xnrpc.commands.upload:main",
        #     "register = xnrpc.commands.register:main",
        # ],
        "console_scripts": [
            "xnrpc = xnrpc.__main__:main",
        ],
    },
)
```

解释一下各个配置的含义：

* name -> 为项目名称，和顶层目录名称一致;
* version -> 是项目当前的版本，1.0.0.dev1表示1.0.0版，目前还处于开发阶段
* description -> 是包的简单描述，这个包是做什么的
* long_description -> 这是项目的详细描述，出现在pypi软件的首页上
* url -> 为项目访问地址，我的项目放在github上。
* author -> 为项目开发人员名称
* author_email -> 为项目开发人员联系邮件
* license -> 为本项目遵循的授权许可
* classifiers -> 有很多设置，具体内容可以参考官方文档
* keywords -> 是本项目的关键词，理解为标签
* packages -> 是本项目包含哪些包，使用工具函数自动发现包
* package_data -> 通常包含与包实现相关的文件
* data_files -> 指定其他的一些文件（如配置文件）
* cmdclass -> build或install的时候执行的额外操作
* entry_points -> 可以定义安装该模块后执行的脚本，比如将某个函数作为linux命令

这里面重点说明下面三个：

1. package_data 通常包含与包实现相关的文件。打包的时候会自动包括进去
2. data_files 指定其他的一些文件（如配置文件并放置在指定的目录），
   如果目录名是相对路径，则是相对于`sys.prefix`或`sys.exec_prefix`的路径
3. cmdclass build或install的时候执行的额外操作

还有一个文件`MANIFEST.in`定义了打源码包的时候需要包含的文件，一个示例如下：

```
include LICENSE
include README.rst
include README.md
include AUTHORS

recursive-include tests *.py
recursive-include etc *.conf
```

## 项目打包

```bash
cd xnrpc/
python setup.py sdist bdist_wheel
```

如果报错：`invalid command 'bdist_wheel'`，则先安装下wheel模块：

```bash
pip install wheel
```

执行完后，在顶层项目目录下将产生几个新的目录：

![](https://xnstatic-1253397658.file.myqcloud.com/pysetup002.png)

## 注册PyPI帐号

如果没有账号需要先在PyPI网站上注册账号。
在您的本机用户下创建~/.pypirc文件，此文件中配置PyPI访问地址和账号。下面是我的.pypirc文件内容请根据自己的账号来修改。

```
[distutils]
index-servers = pypi

[pypi]
repository=http://pypi.python.org/pypi
username=yidao620c
password=********
```

## 注册项目

```bash
python setup.py register
```

如果报错：

```
Server response (403): Must access using HTTPS instead of HTTP
```

解决方法：

使用<https://github.com/pypa/twine>

```bash
pip install twine
```

注册项目:

```bash
twine register dist/xnrpc-1.0.0.tar.gz
twine register dist/xnrpc-1.0.0-py2-none-any.whl
```

通过上面`.pypirc`文件中的配置，在PyPI上注册项目信息，成功注册之后，可以在PyPI上看到自己的项目名称：

## 上传项目

```bash
# python setup.py sdist bdist_wheel upload
# 安装了twine使用
twine upload dist/*
```

通过上面`.pypirc`文件中的配置，上传打包文件，可以在PyPI上看到上传的项目文件：

![](https://xnstatic-1253397658.file.myqcloud.com/pysetup003.png)

## 安装测试

恭喜你成功将你的软件包上传至PyPI上面，全世界的人都可以通过pip来安装了：

```bash
pip install xnrpc
```

## 下载量分析

安装:

```bash
pip install vanity
```

使用:

```
vanity xnrpc
vanity xnrpc==1.0.0
```
