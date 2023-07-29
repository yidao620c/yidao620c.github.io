---
title: 使用Nuitka打包Python程序
date: 2020-05-12 22:22:26 +0800
toc: true
categories: [ python ]
tags: [ python, nuitka ]
---
首先通过PyQt5来创建一个GUI程序，比如我写的一个简单的计算器程序目录如下：
![img.png](https://xnstatic-1253397658.file.myqcloud.com/20200512-01.png)

运行效果如下：
![img.png](https://xnstatic-1253397658.file.myqcloud.com/20200512-02.png)

需要将这个GUI打包成windows上面的exe文件。发现了2个都能对python项目打包的工具：pyintaller和nuitka。

这2个工具同时都能满足项目的需要，两者都具备的优点：

1. 隐藏源码。这里的pyinstaller是通过设置key来对源码进行加密的；
   而nuitka则是将python源码转成C++（这里得到的是二进制的pyd文件，防止了反编译），然后再编译成可执行文件。
2. 方便移植。用户使用方便，不用再安装什么python啊，第三方包之类的。

<!--more-->

2个工具使用后的最大的感受就是：

pyinstaller体验很差！

* 一个深度学习的项目最后转成的exe竟然有近3个G的大小（pyinstaller是将整个运行环境进行打包），对，你没听错，一个EXE有3个G！
* 打包超级慢，启动超级慢。

nuitka真香！

* 同一个项目，生成的exe只有7M！
* 打包超级快（1min以内），启动超级快。

## Nuitka的安装

* 直接利用pip即可安装：`pip install Nuitka`
* 安装zstandard：`pip install zstandard`
* 下载MinGW64，最低需要11.2版，只能去官网github上面下载了。

MinGW64下载地址：https://github.com/brechtsanders/winlibs_mingw/releases/download/

下载文件名：`winlibs-x86_64-posix-seh-gcc-11.2.1-snapshot20211211-mingw-w64ucrt-9.0.0-r1.7z`

下载后解压缩，然后将bin目录加入到系统环境变量的PATH中去。最后开cmd命令窗口，执行`gcc -v`查看版本号显示成功则没问题了。

同时还要下载ccache，解压后将ccache.exe放到某个目录，并且将此目录也放到PATH中去。

同时还要下载https://dependencywalker.com/，也是将其解压缩到某个目录，并且将此目录放到PATH中去。

## 开始打包
对于第三方依赖包较多的项目（比如需要import torch,tensorflow,cv2,numpy,pandas,geopy等等）而言，
这里最好打包的方式是只将属于自己的代码转成C++，不管这些大型的第三方包！

常用命令的参数解释：

* --mingw64 #默认为已经安装的vs2017去编译，否则就按指定的比如mingw
* --standalone 如果是为了分发文件夹dist，文件夹下带上依赖包使用此选项。
* --onefile 独立文件，如果想将所有依赖打成一个包，选这个。
* --windows-disable-console 没有CMD控制窗口
* --output-dir=out 生成exe到out文件夹下面去
* --show-progress 显示编译的进度，很直观
* --show-memory 显示内存的占用
* --plugin-enable=pylint-warnings 报警信息
* --plugin-enable=qt-plugins 需要加载的PyQT插件
* --nofollow-imports # 所有的import不编译，交给python3x.dll执行
* --windows-icon-from-ico=./logo.ico：指定生成的exe的图标为logo.ico这个图标，这里推荐一个将图片转成ico格式文件的网站（比特虫）。
* --follow-import-to=need # need为你需要编译成C/C++的py文件夹命名

使用以下命令（调试）直接生成exe文件：
```
nuitka --onefile --windows-disable-console --mingw64 --nofollow-import-to=tensorflow,numpy --show-memory --show-progress --plugin-enable=qt-plugins --output-dir=out --windows-icon-from-ico=./logo.ico main.py
```

经过1min的编译之后，你就能在你的out目录下看到如下3个文件夹+1个目标可执行文件`main.exe`，三个文件夹都可以删了。只保留这个exe文件即可。
![img.png](https://xnstatic-1253397658.file.myqcloud.com/20200512-04.png)

当然这里你会发现真正运行exe的时候，会报错：`no module named torch,cv2,tensorflow`等等这些没有转成C++的第三方包。

这里需要找到这些包（我的是在python3.8\Lib\site-packages下）复制（比如numpy,cv2这个文件夹）到main.dist路径下。

至此，exe能完美运行啦！