---
title: 每天5分钟玩转Python（21） - 类和实例
date: '2019-06-21 10:23:54 +0800'
toc: true
categories: [ python ]
tags: [ python教程 ]
---

面向对象编程（OOP）跟面向过程编程是两种程序设计思想，OOP将计算机程序视为一组对象的集合，
这些对象直接可通过发送消息来通信，程序执行的就是这一系列的消息。而面向过程将程序视为一组命令或函数集合，
函数又划分为多个子函数以降低系统复杂度。

Python既支持面向过程编程，又支持面向对象编程。封装、继承和多态是面向对象编程的三大特点。
面向过程就是以函数为中心的编程，这个在前面已经讲过，从这篇开始正式讲解Python面向对象编程。
<!-- more -->

## 类和实例

面向对象编程中最重要的两个概念是类（Class）和实例（Instance），实例有时候也可称为对象（Object）。
由于实例这个词更能体现其本质含义，后面我都用实例（Instance）来表述。类就是一个抽象模板，
而实例是这个抽象模板的具体实现，这个就是为了模拟现实世界的，比较好理解。比如手机是一个类，
而在双十一抢购的某一个华为P30手机就是一个手机的实例。

我们以手机为例子来说明一下类和实例的使用。在Python中，定义类是通过class关键字：

```python
class Phone(object):
    pass
```

这里定义了一个Phone类，从object继承，继承的概念后面会讲。

## 类的属性

前面讲过类的本质就是一个抽象模板，所以我们可以在这个模板中定义类必须有的属性。
比如手机会有品牌、型号、重量、机身颜色等属性。可以通过一个特殊的初始化方法`__init__`来讲这些属性绑定上去：

```python
class Phone(object):
    def __init__(self, brand, model, weight, color):
        self.brand = brand
        self.model = model
        self.weight = weight
        self.color = color
```

注意到`__init__`方法的第一个参数永远是self，代表的是实例本身。

通过调用构造函数，传入必要的参数，就能初始化一个手机实例出来：

```python
phone = Phone('Huawei', 'P30', '174.00g', '幻影蓝')
```

注意调用类的实例化方法的时候，不需要传递第一个`self`参数。

## 类的方法

之前讲过OOP中程序执行就是通过实例间发送消息来实现的，而发送消息就是调用实例的方法。
比如我们在Phone这个类中定义一个展示手机颜色的方法：

```python
class Phone(object):
    def __init__(self, brand, model, weight, color):
        self.brand = brand
        self.model = model
        self.weight = weight
        self.color = color

    def print_color(self):
        print('color is {}'.format(self.color))
```

测试一下这个方法：

```python
phone = Phone('Huawei', 'P30', '174.00g', '幻影蓝')
phone.print_color()
```

注意方法调用也不需要传递第一个`self`参数。

这样做的好处是我们只需要给这个类提供初始化的属性值，至于打印颜色是如何打印的客户端不需要关注里面的细节。
这就是面向对象中的封装。可以根据需要为类增加多个方法，完成不同的功能。

## 访问限制

虽说我们通过方法将内部的实现细节隐藏起来，但是外部代码可以直接修改实例的属性值。比如上面的weight、color等。

```python
phone = Phone('Huawei', 'P30', '174.00g', '幻影蓝')
phone.color = 'blue'
```

有时候我并不想外部直接访问我的属性，只能通过方法来执行修改动作。这样我就能在方法中控制外部对类的操作，实现可控。

我们可以将实例的变量前面加两个下划线`__`。在python中如果遇到变量以`__`开头，则认为是私有变量。只能内部访问到。

```python
class Phone(object):
    def __init__(self, brand, model, weight, color):
        self.__brand = brand
        self.__model = model
        self.__weight = weight
        self.__color = color

    def print_color(self):
        print('color is {}'.format(self.__color))

phone = Phone('Huawei', 'P30', '174.00g', '幻影蓝')
phone.print_color()
print(phone.__color)
```

运行后报错：

```
AttributeError: 'Phone' object has no attribute '__color'
```

如果想让外部访问/修改私有属性，则可以增加响应的方法：

```python
class Phone(object):
    # ...前面省略
    
    def get_color(self):
        return self.__color
    
    def set_color(self, color):
        self.__color = color
```

还有一点要注意，在python中有很多内置的变量或方法名是同时以双下划线开头和结尾，比如上面的`__init__`，
所以我们定义的私有变量名称不要同时以双下划线开头和结尾。

是否我定了一个私有变量`__color`就一定不能直接访问了呢？答案是还可以访问，
只不过访问的方式变成`phone._Phone__color`。就是在变量前面加上下划线和类名称。虽然你可以这样做，
但是强烈建议你不要去搞事情。Python本身不会限制你做坏事，一切得靠自觉。

