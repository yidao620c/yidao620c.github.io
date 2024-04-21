---
title: 每天5分钟玩转Python（23） - 对象属性
date: '2019-06-23 16:46:22 +0800'
toc: true
categories: [ python ]
tags: [ python教程 ]
---

前面介绍了面向对象编程的三个基本特性：封装、继承和多态。这一章开始讲解一些高级特性，可以让我们如虎添翼，
编写更高阶的程序。包括对象属性、多继承、定制类和元类。
<!-- more -->

## 使用__slots__

当我们创建了一个类的实例后，可以给这个实例绑定任何属性和方法，这是动态语言的灵活性。

比如我们定义了一个Phone类：

```python
class Phone(object):
    pass
```

我们创建一个实例，然后给它绑定新的属性和方法：

```python
p = Phone()
p.name = 'Xiong Neng'  # 绑定属性
print(p.name)

def set_age(self, age): # 定义一个函数作为实例方法
    self.age = age
from types import MethodType
p.set_age = MethodType(set_age, p) # 给实例绑定一个方法
p.set_age(25) # 调用实例方法
print(p.age) # 测试结果
```

这里要注意，给实例绑定的方法和属性只对这个实例有用，对其他实例不起作用。

有些时候我们想限制实例绑定的属性该咋办捏？比如我只允许Phone的实例拥有brand和color属性。
Python允许我们定义class的时候定义一个特殊的`__slots__`变量，来限制该class实例能添加的属性：

```python
class Phone(object):
    __slots__ = ('brand', 'color') # 用tuple定义允许绑定的属性名称

p = Phone()
p.name = 'Xiong Neng'  # 绑定属性
print(p.name)
```

运行后报错`AttributeError: 'Phone' object has no attribute 'name'`。

`__slots__`定义的属性仅对当前类实例起作用，对子类不起作用。
除非在子类中也定义`__slots__`，这样，子类实例允许定义的属性就是自身的`__slots__`加上父类的`__slots__`。

## 使用@property

在绑定属性时，如果我们直接把属性暴露出去，虽然写起来很简单，但是没办法进行参数检查。
比如在设置手机颜色的时候，只能设置红、蓝、白三种颜色。可以通过增加方法来检查：

```python
class Phone(object):
    def __init__(self, brand, color):
        self._brand = brand
        self._color = color

    def get_color(self):
        return self._color

    def set_color(self, value):
        if not isinstance(value, str):
            raise ValueError('color must be an string!')
        if value not in ('red', 'blue', 'white'):
            raise ValueError('score must be one of red, blue, white!')
        self._color = value
```

但是调用方法的方式显得比较繁琐。不如直接写属性方式简单直接。对于追求完美的pythoner来讲得两者兼顾才爽。
还记得之前讲的装饰器（decorator）可以给函数动态加上功能吗？对于类的方法，装饰器一样起作用。
Python内置的`@property`装饰器就是负责把一个方法变成属性调用的：

```python
class Phone(object):
    def __init__(self, brand, color):
        self._brand = brand
        self._color = color

    @property
    def color(self):
        return self._color

    @color.setter
    def color(self, value):
        if not isinstance(value, str):
            raise ValueError('color must be an string!')
        if value not in ('red', 'blue', 'white'):
            raise ValueError('score must be one of red, blue, white!')
        self._color = value


p = Phone('brand', 'red')
p.color = 'blue'
```

`@property`的实现比较复杂，我们先考察如何使用。把一个getter方法变成属性，只需要加上`@property`就可以了，
此时`@property`本身又创建了另一个装饰器`@score.setter`，负责把一个setter方法变成属性赋值，
于是，我们就拥有一个可控的属性操作。还有一点要注意是两个方法的名称就是属性名，要统一。

`@property`广泛应用在类的定义中，可以让调用者写出简短的代码，同时保证对参数进行必要的检查，
这样，程序运行时就减少了出错的可能性。


