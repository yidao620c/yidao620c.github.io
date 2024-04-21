---
title: 每天5分钟玩转Python（01） - 入门简介
date: '2019-06-01 12:30:20 +0800'
toc: true
categories: [ python ]
tags: [ python教程 ]
---

人生苦短，我用Python！

终于要写这个系列教程了，虽然我知道会很辛苦，也很难写的比较完美。总是有其他的事情干扰，
不过我有个特点就是一旦开始就停不下来，我相信自己会坚持写完这个入门教程的。

市面上有好多Python入门书籍，还有各种培训课程上面的教程，可能有人问我为啥还要写这个玩意。
我的解释是，总会有那么一小部分人看得懂我在写啥，喜欢这种风格，就足够了。
<!-- more -->

之前看过《每天5分钟玩转docker容器》和《每天5分钟玩转kubernetes》两个系列，我比较喜欢他这种风格。
所以基本上跟那个类似，从易到难，一点点突破，每天只需要花费几分钟时间。日拱一卒就可以了。最后能坚持看完的，
一定会有收获。或者也算是对自己的一种挑战吧。人总是要有点挑战才能进步吧！

## Python历史

Python的创始人叫Guido van Rossum，之所以取这个名字是因为他是一个叫"Monty Python"的喜剧团体的爱好者。
Python诞生于1989年，作者曾经是一个C++程序员，参加设计过ABC的教学语言，他觉得这个语言很优雅，使用简单，
天生就是为非专业程序员设计的。但是ABC语言由于过于封闭导致也没普及开来。Guido大师决定自己搞一个，参考ABC语言，
取其精华去其糟粕，然后就有了Python这门语言。

目前这门语言热度在TIOBE排行榜上稳居第三，已经超过了C++语言。

![](https://xnstatic-1253397658.file.myqcloud.com/py5_tiobe.png)

## Python应用领域

Python应用领域非常广泛，我列一下目前比较热门的应用领域。

### WEB开发

Python的Web开发框架逐渐成熟，比如耳熟能详的Django和Flask, 你可以快速地开发功能强大的Web应用。
个人首推Flask，强烈建议所以有志于从事Python Web开发的人掌握这门框架。无论是建大型网站，
开发OA或Web API，Flask都可以轻松胜任。

### 网络爬虫

使用python，几行代码搞定爬虫，可以节省大量时间，能够编写网络爬虫的编程语言有不少，但Python绝对是其中的主流之一。
Python自带的urllib库，第三方的requests库和Scrappy框架让开发爬虫变得非常容易。

### 科学计算和数据分析

随着NumPy，SciPy，Matplotlib等众多程序库的开发和完善，Python越来越适合于做科学计算和数据分析了。
它不仅支持各种数学运算，还可以绘制高质量的2D和3D图像。和科学计算领域最流行的商业软件Matlab相比，
Python比Matlab所采用的脚本语言的应用范围更广泛，可以处理更多类型的文件和数据。

### 人工智能

这个就不需要我多说了，将来的十年必定是AI的天下，现在AI的热度已经席卷全球。
Python在人工智能大范畴领域内的机器学习、神经网络、深度学习等方面都是主流的编程语言，
得到广泛的支持和应用。最流行的神经网络框架如Facebook的PyTorch和Google的TensorFlow都采用了Python语言。
你不学Python, 你会用那些框架吗?

### 自动化运维

运维工程师最喜欢使用的脚本语言，很多运维工具都是用python实现，一般说来，
Python编写的系统管理脚本在可读性、性能、代码重用度、扩展性几方面都优于普通的shell脚本。

### 云计算

Python的最强大之处在于模块化和灵活性，而构建云计算的平台的IasS服务的OpenStack就是采用Python的，
云计算的其他服务也都是在IasS服务之上的。

### 网络编程

Python提供了丰富的模块支持sockets编程，能方便快速地开发分布式应用程序。
很多大规模软件开发计划例如Zope，Mnet, BitTorrent和Google都在广泛地使用它。

### 游戏开发

很多游戏使用C++编写图形显示等高性能模块，而使用Python或者Lua编写游戏的逻辑、服务器。
相较于Python，Lua的功能更简单、体积更小，然而Python则支持更多的特性和数据类型。
Python的PyGame库也可用于直接开发一些简单游戏。

## 编程语言简介和特点

编程语言主要从以下几个角度为进行分类

* 编译型和解释型
* 静态语言和动态语言
* 强类型定义语言和弱类型定义语言

每个分类代表什么意思呢。

**_编译和解释型语言_**

> CPU不能直接认识并执行我们写的语句,它只能认识机器语言(CPU指令集，二进制的形式)。
> 因此我们开发语言的Virtual Machine要将识别的开发语言转换成机器语言让CPU去执行；那么就有两种以下两种方式：
> 1. 编译器是把源程序的每一条语句都编译成机器语言，并保存成二进制文件，这样运行时计算机可以直接以机器语言来运行此程序，速度很快;
> 2. 解释器则是只在执行程序时才一条一条的解释成机器语言给计算机来执行，所以运行速度是不如编译后的程序运行的快的;

 编译型      | 解释型         | 混合型  
----------|-------------|------
 C        | Java Script | Java 
 C++      | Python      | C#   
 GO       | Ruby        | N/A  
 Swift    | PHP         | N/A  
 Ojbect-C | Perl        | N/A  

**_静态和动态语言_**

> 动态类型语言是指在运行期间才去做数据类型检查的语言实现不需要声明变量的数据类型；
> 而静态类型语言的数据类型是在编译其间检查的，也就是说在写程序时要声明所有变量的数据类型。

Python就是一种典型的动态类型语言，C/C++是静态类型语言的典型代表，其他的静态类型语言还有C#、JAVA等。

_**强类型和弱类型定义语言**_

> 强类型定义语言是指一旦一个变量被指定了某个数据类型，如果不经过强制转换，那么它就永远是这个数据类型了。

Python是强类型定义语言。

## Python哲学

可以使用`import this`来打印python哲学。

```
The Zen of Python, by Tim Peters

Beautiful is better than ugly.
Explicit is better than implicit.
Simple is better than complex.
Complex is better than complicated.
Flat is better than nested.
Sparse is better than dense.
Readability counts.
Special cases aren't special enough to break the rules.
Although practicality beats purity.
Errors should never pass silently.
Unless explicitly silenced.
In the face of ambiguity, refuse the temptation to guess.
There should be one-- and preferably only one --obvious way to do it.
Although that way may not be obvious at first unless you're Dutch.
Now is better than never.
Although never is often better than *right* now.
If the implementation is hard to explain, it's a bad idea.
If the implementation is easy to explain, it may be a good idea.
Namespaces are one honking great idea -- let's do more of those!
```

我只列出其中几个：

1. 优雅 - 语法非常的简短干练，没有一点多余的语法结构。
1. 明确 - python对格式进行强制的限制；将格式整齐划一，就感觉在写诗一样优雅美丽；
1. 简单 - 在python的设计哲学中，要实现任何一件事情，都应该有一种并且我们认为是最好的一种方式去实现。
