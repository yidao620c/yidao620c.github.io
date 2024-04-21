---
title: 每天5分钟玩转Python（17） - 模块和包
date: '2019-06-17 12:30:35 +0800'
toc: true
categories: [ python ]
tags: [ python教程 ]
---

模块和包都是用来组织代码用的，在python中一个模块就是一个.py文件，
而一个包就是一个包含了`__init__.py`的文件夹。
使用模块最大的好处就是提高代码可维护性，我们在编写代码的时候通常会引用内置模块或第三方模块。

引入包是为了解决命名冲突问题，你可以把包当成是命名空间，
比如你写的`abc.py`模块和其他人写的`abc.py`模块只要在不同的包中就不会冲突。
只要顶层的包名不与别人冲突，那所有模块都不会与别人冲突。
<!-- more -->

比如我现在有这样一个目录结构：

    winhong/
        web/
            __init__.py
            util.py
        __init__.py   (定义cool()函数)
        main.py       (定义aa()函数和User类)

每一个包目录下面都会有一个`__init__.py`的文件，这个文件是必须存在的，否则，Python就把这个目录当成普通目录，
而不是一个包。`__init__.py`可以是空文件，也可以有Python代码，`__init__.py`本身就是一个模块，
而它的模块名就是所在文件夹名，比如`winhong`目录下的`__init__.py`模块名就是`winhong`，
而web目录下的`__init__.py`的模块名就是`web`。

注：自己编写模块时候

1. 模块名不要跟内置模块名冲突，例如系统自带了`sys`模块，自己的模块就不可命名为`sys.py`
2. 模块内函数与变量名称不要和内置函数名冲突，
   点击<http://docs.python.org/3/library/functions.html>查看Python的所有内置函数

我截了个最新的内置函数图：

![](https://xnstatic-1253397658.file.myqcloud.com/py17_builtin.png)

## 导入模块

要使用模块就需要先导入

```python
#!/usr/bin/env python
# -*- coding: utf-8 -*-
"""
这里是模块说明
"""
__author__ = "Xiong Neng"

import sys
def demo():
    print('\n'.join(sys.path))

if __name__ == '__main__':
    demo()
```

上面第一行是解释器路径，这个是标准写法，第二行指定源文件编码utf-8，这样可支持中文。
接下来是模块文档注释，任何模块代码的第一个字符串都被视为模块的文档注释。
下面的`__author__ = "Xiong Neng"`指定了作者，你公开源码后别人可看到。
接下来是主体部分，这个是一个模块的标准写法。

注意到：

```python
if __name__ == '__main__':
    demo()
```

当我们直接运行这个模块时候，Python解释器把一个特殊变量`__name__`置为`__main__`，
而如果在其他地方导入该`hello`模块时，if判断将失败，所以这个是模块测试的标准写法。

## 模块搜索路径

当我们试图加载一个模块时，Python会在指定的路径下搜索对应的.py文件，如果找不到就会报错。
默认情况下，Python解释器会搜索当前目录、所有已安装的内置模块和第三方模块，搜索路径存放在sys模块的path变量中：

```python
import sys

print('\n'.join(sys.path))
```

如果我们要添加自己的搜索目录，有两种方法：

第一种方法是直接修改`sys.path`，添加要搜索的目录，这种方法是在运行时修改，运行结束后失效：

```python
import sys

sys.path.append('/Users/xiongneng/my_module_dir')
```

第二种方法是设置环境变量`PYTHONPATH`，该环境变量的内容会被自动添加到模块搜索路径中。
设置方式与设置Path环境变量类似。注意只需要添加你自己的搜索路径，Python自己本身的搜索路径不受影响。

## 私有属性

一般我们会在模块中定义很多函数和变量，但是我们只想公开一部分，那么我们可使用单下划线_来实现。

类似`__xxx__`这样的变量是特殊变量，可以被直接引用，但是有特殊用途，
比如上面的`__author__`，`__name__`就是特殊变量，我们没事就别在模块中定义这些东西了。

在模块中：

1. 单下划线`_`和双下划线`__`开头的变量表示这个属性和函数是私有的，不应该直接访问
3. 对于`from module import *`这样的导入，不管单下划线还是双下划线都不会被导入
4. 对于`import module`这样的导入都可以使用，但不建议使用，PEP8会告警调用单下划线_

总的来说就是，Python本身没有任何机制阻止你干坏事，一切全靠约定和自觉。

## 特殊属性

对于模块、包、类、对象，函数等，它们都有各自一些特殊的内置属性值，我这里选几个比较有用的讲一下：

1. `__package__`表示包名
2. `__module__`表示模块名
3. `__name__`表示名字
4. `__file__`表示py文件路径
5. `__dir__`是该对象中所有属性集合

现在我有这样一个结构：

    samples/
        __init__.py
            def cool(): pass
        main.py
            def aa(): pass
            class User(object): pass

然后我来对这些对象做一个测试，注意我注释掉的是不支持属性并给出报错信息：

```python
# 打印python解释器路径和PYTHONPATH
import sys

print(sys.executable)
print('------------------------分割线----------------------------')

print('\n'.join(sys.path))
print('------------------------分割线----------------------------')

# aa是一个模块函数
from samples.main import aa
print("type={}".format(type(aa)))
# AttributeError: 'function' object has no attribute '__package__'
# print("aa.__package__={}".format(aa.__package__))
print("aa.__module__={}".format(aa.__module__))
print("aa.__name__={}".format(aa.__name__))
# AttributeError: 'function' object has no attribute '__file__'
# print("aa.__file__={}".format(aa.__file__))
print('------------------------分割线----------------------------')

# main是一个模块
import samples.main as main
print("type={}".format(type(main)))
print("init.__package__={}".format(main.__package__))
# AttributeError: 'module' object has no attribute '__module__'
# print("init.__module__={}".format(main.__module__))
print("init.__name__={}".format(main.__name__))
print("init.__file__={}".format(main.__file__))
print('------------------------分割线----------------------------')

# samples是一个模块，同时也是一个包
import samples
print("type={}".format(type(samples)))
print("samples.__package__={}".format(samples.__package__))
# AttributeError: 'module' object has no attribute '__module__'
# print("samples.__module__={}".format(samples.__module__))
print("samples.__name__={}".format(samples.__name__))
print("samples.__file__={}".format(samples.__file__))
print('------------------------分割线----------------------------')

# 导入包samples的__init__模块看看
import samples.__init__ as sample_init
print("type={}".format(type(sample_init)))
print("sample_init.__package__={}".format(sample_init.__package__))
# AttributeError: 'module' object has no attribute '__module__'
# print("sample_init.__module__={}".format(sample_init.__module__))
print("sample_init.__name__={}".format(sample_init.__name__))
print("sample_init.__file__={}".format(sample_init.__file__))
print('------------------------分割线----------------------------')

# 最后打印samples和sample_init的dir()，看看是否不一样
print(dir(samples))  # 多了__init__，__path__，main这三个属性
print(dir(sample_init))
print('------------------------分割线----------------------------')

# 类的导入
from samples.main import User
print("type={}".format(type(User)))
# AttributeError: type object 'User' has no attribute '__package__'
# print("User.__package__={}".format(User.__package__))
print("User.__module__={}".format(User.__module__))
print("User.__name__={}".format(User.__name__))
# AttributeError: type object 'User' has no attribute '__file__'
# print("User.__file__={}".format(User.__file__))

print('------------------------分割线----------------------------')
# 对象
user = User("XiongNeng")
print("type={}".format(type(user)))
# AttributeError: 'User' object has no attribute '__package__'
# print("user.__package__={}".format(user.__package__))
print("user.__module__={}".format(user.__module__))
# AttributeError: 'User' object has no attribute '__name__'
# print("user.__name__={}".format(user.__name__))
# AttributeError: 'User' object has no attribute '__file__'
# print("user.__file__={}".format(user.__file__))

print('------------------------分割线----------------------------')
```
