---
title: 使用PyCharm进行远程开发和调试
date: 2016-05-26 10:02:42 +0800
toc: true
categories: [ 开发工具 ]
tags: [ 开发工具 ]
---

你是否经常要在Windows 7或MAC OS X上面开发Python或Web应用程序，但是它们最后需要在linux上面来运行呢？
我们经常会碰到开发时没有问题但是到了正式的Linux环境下面却出现问题。那么怎样保证开发环境跟运行环境的一致呢？

通常有两种方法解决。一种是使用PyCharm内置支持的`Vagrant`，
这个教程可以参考[Vagrant开发环境配置](https://github.com/astaxie/Go-in-Action/blob/master/ebook/zh/01.0.md)。

不过很遗憾的是我自己在试验过程中启动VirtualBox虚拟机时候老是报错，暂时还没解决，读者可以自己试着测试看行不行。
第二种方式就是通过PyCharm的远程解释器加上文件同步功能，实现本地编辑代码->同步到服务器->通过远程debug来调试测试程序。
目前我选择的是第二种，虽然比第一种更笨拙点。

远程调试的功能在Eclipse、IntelliJ IDEA等大型IDE中均有支持，实现原理都基本相同，这里采用PyCharm进行说明。
<!-- more -->

### 远程服务器的同步配置

远程服务器IP地址192.168.203.95，开启ssh服务，安装python版本2.7。我用一个在PyCharm里面的core-python项目来做演示。

首先我们需要配置PyCharm通服务器的代码同步，打开Tools | Deployment | Configuration

点击左边的"+"添加一个部署配置，输入名字，类型选SFTP
![](https://xnstatic-1253397658.file.myqcloud.com/pcr001.png)

确定之后，再配置远程服务器的ip、端口、用户名和密码。root path是文件上传的根目录，注意这个目录必须用户名有权限创建文件。
![](https://xnstatic-1253397658.file.myqcloud.com/pcr002.png)

然后配置映射，local path是你的工程目录，就是需要将本地这个目录同步到服务器上面，我填的是项目根目录。
Deploy path on server 这里填写相对于root path的目录，下面那个web path不用管先
![](https://xnstatic-1253397658.file.myqcloud.com/pcr003.png)

如果你还有一些文件或文件夹不想同步，那么在配置对话框的第三个tab页"Excluded path"里面添加即可，可同时指定本地和远程。

还有一个设置，打开Tools | Deployment | Options，将"Create Empty directories"打上勾，要是指定的文件夹不存在，会自动创建。

### 上传和下载文件

有几种方法可以实现本地和远程文件的同步，手动和当文件保存后自动触发。这里我选择了手动，因为自动触发比如影响性能，PyCharm会卡，感觉不爽。

手动上传方式很简单，选择需要同步的文件或文件夹，然后选择 Tools | Deployment | Upload to sftp(这个是刚刚配置的部署名称)
![](https://xnstatic-1253397658.file.myqcloud.com/pcr004.png)

下载文件也是一样，选择 Tools | Deployment | Download from sftp

### 比较远程和本地文件

有时候你并不确定远程和本地版本的完全一致，需要去比较看看。PyCharm提供了对比视图来为你解决这个问题。

选择Tools | Deployment | Browse Remote Host，打开远程文件视图，在右侧窗口就能看到远程主机中的文件
![](https://xnstatic-1253397658.file.myqcloud.com/pcr005.png)

选择一个你想要对比的文件夹，点击右键->Sync with Local，打开同步对比窗口，使用左右箭头来同步内容。

上面是服务器与本地对比，那么本地文件通服务器对比，就先在PyCharm里面选择文件或文件夹，然后右键->Deployment->Sync with
deployed to即可

### PyCharm远程调试

在PyCharm中进行远程调试有两种选择：

1. 使用远程的解释器
2. 使用Python调试服务器

这里简单起见我只演示第一种，使用远程解释器，也就是使用服务器上面安装的python解释器。

#### 配置远程Python解释器

选择File | Settings，选择Project | Project Interpreter，然后在右边，点击那个小齿轮设置，如下
![](https://xnstatic-1253397658.file.myqcloud.com/pcr006.png)

然后点击"Add Remote"，填写主机的ssh配置
![](https://xnstatic-1253397658.file.myqcloud.com/pcr007.png)

如果之前配置过SFTP的话就直接选"Deployment configuration"，然后选择刚刚的模板名称就可以了，由于我上面配置过就直接选模板，
这里请仔细看我的Python解释器是虚拟环境virtualenv，这个要在服务器上面先创建好虚拟环境。
![](https://xnstatic-1253397658.file.myqcloud.com/pcr009.png)

#### 开始调试

完成之后选择这个远程的解释器作为工程的解释器即可,然后配置一个运行实例，打断点调试。
这里我以另外一个django工程为例来说明，名字为zspace，因为用一个web工程来说明更具代表性。

选择"Run/Debug Configuration"，添加一个"Django server"，然后配置像下面这样写
![](https://xnstatic-1253397658.file.myqcloud.com/pcr010.png)
请注意图中标出的几个点，具体什么意思就不用多解释了吧，^_^

然后你就可以像本地调试一样打断点做调试了。这个步骤太简单就不截图了，记得修改源码后同步到服务器继续下一次的调试。

