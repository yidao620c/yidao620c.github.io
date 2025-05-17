---
title: 使用PyInstaller打包Python程序
date: 2022-05-01 10:22:26 +0800
toc: true
categories: [ python ]
tags: [ python ]
---

PyInstaller是一个能将Python程序转换成单个可执行文件的程序，
操作系统支持Windows, Linux, Mac OS X, Solaris和AIX。并且很多包都支持开箱即用，不依赖环境。

环境为windows7操作系统，python2.7.8 virtual environment

官网：<https://github.com/pyinstaller/pyinstaller>
<!-- more -->

### 详细步骤：

1\. win7下面先安装这个依赖：pywin32，下载下来后切换到venv2.7，然后使用easy_install xxx.exe安装

2\. pip安装PyInstaller：

```
pip install pyinstaller
```

3\. 打包过程中可能会出现msvcp90.dll找不到的问题，
去<http://cn.dll-files.com/msvcp90.dll.html>下载第三个zip文件，
解压后放到C:\Windows\System32，如果是64位的还要放到C:\Windows\SysWOW64目录下。

4\. 再次运行报MSVCR90.dll找不到，同理去<http://cn.dll-files.com/MSVCR90.dll.html>下载MSVCR90.dll，
放到C:\Windows\System32和C:\Windows\SysWOW64中。

5\. 将你的整个程序先复制到某个临时文件夹下面，比如D:\tmp\core-wxpython，此目录下有个main.py是执行入口

6\. 执行build命令，并添加必要的搜索路径，外加执行文件的图标：

```
cd D:\tmp\core-wxpython
pyinstaller -F -w -i d:\tmp\main.ico main.py
```

如果还想添加自定义的依赖库，就要加上-p参数：

```
pyinstaller -F -w -p D:\tmp\core-python\libs -i d:\tmp\main.ico main.py
```

参数说明：

```
-F 表示生成单个可执行文件
-w 表示去掉控制台窗口，这在GUI界面时非常有用。不过如果是命令行程序的话那就把这个选项删除吧！
-p 表示你自己自定义需要加载的类路径，一般情况下用不到
-i 表示可执行文件的图标
```

*注意的事情*

1). 检查生成的\XXX\build\pyi.win32\XXX\warnXXX.txt(XXX是你的项目名）中，
是否缺少了必要的模块。如果有缺少的，那么去如上所述，添加必要的搜素路径，
使得pyinstaller在运行时，可以找到对应的模块并集成进来。

2). 此处我这里没有UPX，暂时没去折腾。估计是用UPX去压缩，压缩后所生成的exe文件的大小，会小得多。

7\. 如果发现报错：pywintypes.error: (193, ‘LoadLibraryEx’… )
原因是添加图标后缀必须是xxx.ico才行，重新去网上下载一个ico格式的图片，再次运行就好了。

8\. 我测试了一个使用wxpython写的gui程序，源码里面引用了一张图片，
使用wx.Image(os.path.abspath(__file__/me.jpg), wx.BITMAP_TYPE_JPEG)来加载，
然后打包成exe后发现找不到图片了，报错。

解决办法：

第一步，在程序中将资源文件都放到一个单独的文件夹中，比如项目根目录下面的resources

第二步，修改程序中引用这些资源文件比如图片的代码：

```python
def resource_path(relative_path):
    """定义一个读取相对路径的函数"""
    if hasattr(sys, "_MEIPASS"):
        base_path = sys._MEIPASS
    else:
        base_path = os.path.abspath(".")
    return os.path.join(base_path, relative_path)
```

然后每次在获取图片的时候，这么引用它的目录：

```python
img = wx.Image(resource_path('resources/me.jpg'), wx.BITMAP_TYPE_JPEG)
```

第三步，先运行第6步生成一个main.spec文件

第四步，修改main.spec文件：

```python
# -*- mode: python -*-
a = Analysis(['main.py'],
             pathex=['d:\\tmp\\core-wxpython'],
             hiddenimports=[],
             hookspath=None,
             runtime_hooks=None)
pyz = PYZ(a.pure)
exe = EXE(pyz,
          a.scripts,
          a.binaries,
          a.zipfiles,
          a.datas,
          [('\\resources\\me.jpg', 'D:\\tmp\\core-wxpython\\resources\\me.jpg', 'DATA')],
          name='main.exe',
          debug=False,
          strip=None,
          upx=True,
          console=True, icon='d:\\tmp\\main.ico')
```

注意：我在a.datas下面添加了那行配置，具体的路径自己去修改下。

上面是添加单个文件，如果有多个文件，可以一个个的添加。不过如果文件多了话，那么就使用下面的方法：

```python
# -*- mode: python -*-
a = Analysis(['main.py'],
             pathex=['d:\\tmp\\core-wxpython'],
             hiddenimports=[],
             hookspath=None,
             runtime_hooks=None)

def extra_datas(mydir):
    def rec_glob(p, files):
        import os
        import glob
        for d in glob.glob(p):
            if os.path.isfile(d):
                files.append(d)
            rec_glob("%s/*" % d, files)
    files = []
    rec_glob("%s/*" % mydir, files)
    extra_datas = []
    for f in files:
        extra_datas.append((f, f, 'DATA'))

    return extra_datas

# append the 'resources' dir
a.datas += extra_datas('resources')

pyz = PYZ(a.pure)
exe = EXE(pyz,
          a.scripts,
          a.binaries,
          a.zipfiles,
          a.datas,
          name='main.exe',
          debug=False,
          strip=None,
          upx=True,
          console=True , icon='d:\\tmp\\main.ico')
```

它会把某个指定的文件下的所有文件递归的添加到最终的包中。省去很多事情！

第五步，执行：pyinstaller D:\tmp\core-wxpython\main.spec

然后就大功告成了！！！在dist目录下面有个main.exe单独的可执行文件，打开它吧。^_^

如果在执行过程中出错，或者双击打开没任何反应。
可以先去掉-w参数后，在控制台窗口打开这个可执行文件，会输出详细出错信息去调试。

### 其他问题记录

1\. 找不到pkg_resources

```
ImportError: No module named pkg_resources
```

解决办法是在安装pycrypto之前，先安装distribute库

```bash
curl https://svn.apache.org/repos/asf/oodt/tools/oodtsite.publisher/trunk/distribute_setup.py | python
```

然后再安装windows下面对应的pycrypto库

```
# http://www.voidspace.org.uk/python/modules.shtml#pycrypto
easy_install http://www.voidspace.org.uk/downloads/pycrypto26/pycrypto-2.6.win-amd64-py2.7.exe
```

2\. 打包时加上-w选项去掉console时出错

不要在程序中使用任何print语句，或者是你将stdout重定向到一个日志、文件或任何其他非控制台地方。

最好的方法是利用日志功能，将输出定向到日志文件中去，在main函数开头添加如下代码：

```python
import logging
import tempfile
logging.basicConfig(level=logging.INFO,
                    filename=tempfile.TemporaryFile().name,
                    format='%(asctime)s %(message)s')
```

用到logging的时候，需要配置日志到文件中，而不是console：

```python
import logging

_LOGGING = logging.getLogger(__file__)
```

3\. pyinstaller用one file方式打包的程式如果有用到subprocess.Popen會有問題

问题参考：<http://www.pyinstaller.org/ticket/597>

最後找到的方法是
<http://code.activestate.com/recipes/578300-python-subprocess-hide-console-on-windows/>

建立一個隐藏窗口，就正常了~

最后用pyinstaller設one folder & no console打包都不跳出小窗口了

解决办法就是自定义一个subprocess_call函数来代替subprocess的call调用，不适用Popen了：

```python
def subprocess_call(*args, **kwargs):
    # also works for Popen. It creates a new *hidden* window,
    # so it will work in frozen apps (.exe).
    if IS_WIN32:
        _LOGGING.info('subprocess_call==IS_WIN32')
        startupinfo = subprocess.STARTUPINFO()
        startupinfo.dwFlags = subprocess.CREATE_NEW_CONSOLE | subprocess.STARTF_USESHOWWINDOW
        startupinfo.wShowWindow = subprocess.SW_HIDE
        kwargs['startupinfo'] = startupinfo
    retcode = subprocess.call(*args, **kwargs)
    return retcode
```

调用方法：

```python
exresult = subprocess_call(exe_command, shell=True)
```

这个方法会等命令执行完成，返回值为0表示正常结束！

4\. 打包后不能放到中文路径下执行
解决办法是下载安装PyInstaller的中文支持库，安装后再重新执行pyinstaller打包命令：

```bash
git clone https://github.com/dkw72n/pyinstaller.git
python setup.py install
pyinstaller -F -w -i d:\tmp\main.ico main.py
```
