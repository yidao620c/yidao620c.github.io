---
title: 每天5分钟玩转Python（22） - 继承与多态
date: '2019-06-22 20:09:02 +0800'
toc: true
categories: [ python ]
tags: [ python教程 ]
---

这一篇开始讲解面向对象中最核心的基础知识，包含了继承、多态以及对象的属性访问等。

在面向对象编程中继承是指一个新类的定义基于某个已有的类，新的类叫子类，而比继承的类称为父类或超类。

还是以手机为例，定义一个手机类，拥有品牌、颜色基本属性。

```python
class Phone(object):
    def __init__(self, brand, color):
        self.__brand = brand
        self.__color = color

    def print_color(self):
        print('color is {}'.format(self.__color))
        return self.__color
```

<!-- more -->

然后再定义一个智能手机类，继承自手机类，增加一个触摸屏类型属性：

```python
class SmartPhone(Phone):
    def __init__(self, brand, color, touch_screen):
        super().__init__(brand, color)
        self.__touch_screen = touch_screen
```

继承的好处是子类可以获得父类全部的功能，由于父类Phone里面定义了打印手机颜色的方法`print_color()`，
子类SmartPhone什么都偶不用做就能自动拥有这个方法。

```python
if __name__ == '__main__':
    p = SmartPhone('brand', 'blue', '电容屏')
    p.print_color()
```

当然，子类还是可以增加一些自己的特有方法的。比如可以再SmartPhone里面增加一个打印触摸屏类型的方法：

```python
class SmartPhone(Phone):
    def __init__(self, brand, color, touch_screen):
        super().__init__(brand, color)
        self.__touch_screen = touch_screen
    
    def print_touch_screen(self):
        print('touch_screen is {}'.format(self.__touch_screen))
```

继承还有一个很重要的好处就是可以覆盖重写父类方法。比如我在打印手机颜色的时候，如果是智能机，则前面加上一个前缀。

```python
def print_color(self):
    print('[SmartPhone] '.format(super().print_color()))
```

## 多态

当子类和父类拥有相同的方法，代码运行时总是调用子类的方法，这种情形我们成为多态。
判断一个变量是否是某个类型可以用`isinstance()`来判断：

```python
if __name__ == '__main__':
    p = SmartPhone('brand', 'blue', '电容屏')
    print(isinstance(p, SmartPhone))
    print(isinstance(p, Phone))
```

结果都是True，这说明在继承体系中，如果一个实例的类型是某个子类，那它一定也可以看作是父类。不过反过来就不行了。

多态的好处是当一个方法参数可传入某个父类类型的时候，那么就一定能传入一个子类。
非常适合面向抽象或者是面向接口编程，大家都知道OOP中解耦的关键在于面向接口或抽象编程。
当我们新增加一个子类的时候，无需更改任何依赖抽象父类的代码。符合设计原则中的开闭原则。

## 获取对象类型

当我们拥有一个对象的时候，想要知道这个对象的类型、拥有的方法的时候该咋办？

我可以使用`type()`来判断对象的类型，包括基本数据类型也可以：

```python
if __name__ == '__main__':
    p = SmartPhone('brand', 'blue', '电容屏')
    print(type(p))
    print(type(123))
    print(type('test'))
    print(type(None))
    print(type(type(123)))
```

打印结果如下：

```
<class '__main__.SmartPhone'>
<class 'int'>
<class 'str'>
<class 'NoneType'>
<class 'type'>
```

最后一个结果表明，`type()`函数返回的类型是`type`。

除了基本类型和定义的类外，我们还能判断函数的类型，不过这里需要引入types模块：

```python
import types
def fn():
    pass

print(type(fn) == types.FunctionType)
print(type(abs) == types.BuiltinFunctionType)
print(type(lambda x: x) == types.LambdaType)
print(type((x for x in range(10))) == types.GeneratorType)
```

## 获取对象属性和方法

如果要获得一个对象的所有属性和方法，可以使用内置的`dir()·函数，它返回一个包含字符串的list，
获得一个str对象的所有属性和方法：

```python
print(dir('test'))
```

返回结果

```
['__add__', '__class__', '__contains__', '__delattr__', '__dir__', '__doc__', '__eq__',
 '__format__', '__ge__', '__getattribute__', '__getitem__', '__getnewargs__', '__gt__', 
'__hash__', '__init__', '__init_subclass__', '__iter__', '__le__', '__len__', '__lt__',
 '__mod__', '__mul__', '__ne__', '__new__', '__reduce__', '__reduce_ex__', '__repr__', 
'__rmod__', '__rmul__', '__setattr__', '__sizeof__', '__str__', '__subclasshook__', 
'capitalize', 'casefold', 'center', 'count', 'encode', 'endswith', 'expandtabs', 'find', 
'format', 'format_map', 'index', 'isalnum', 'isalpha', 'isascii', 'isdecimal', 'isdigit',
 'isidentifier', 'islower', 'isnumeric', 'isprintable', 'isspace', 'istitle', 'isupper', 
'join', 'ljust', 'lower', 'lstrip', 'maketrans', 'partition', 'replace', 'rfind', 'rindex', 
'rjust', 'rpartition', 'rsplit', 'rstrip', 'split', 'splitlines', 'startswith', 'strip', 
'swapcase', 'title', 'translate', 'upper', 'zfill']
```

类似`__xx__`的属性和方法在Python中都是有特殊用途的，比如`__len__`方法返回长度。在Python中，
如果你调用`len()`函数试图获取一个对象的长度，实际上，在len()函数内部，它自动去调用该对象的__len__()方法。
这也是为什么我强烈建议你不要把自己的属性也定义成同时以双下划线开头和结尾。

剩下的都是普通属性或方法，比如lower()返回小写的字符串：

## 操作对象属性

仅仅把属性和方法列出来是不够的，配合getattr()、setattr()以及hasattr()，我们可以直接操作一个对象的状态：

```python
p = SmartPhone('brand', 'blue', '电容屏')
print(hasattr(p, 'name'))
print(hasattr(p, '__touch_screen'))  # 双下划线开头的私有属性获取不到
print(hasattr('test', 'isupper'))
print(getattr('test', 'lower'))
setattr(p, 'name', 'ZJJ')  # 设置一个属性'name'
print(getattr(p, 'name', 'nothing'))  # 还能传入一个默认值，如果属性不存在则返回默认值
```

注意最后打印属性还传入了一个默认值，这样可以防止获取不到属性时候跑出的`AttributeError`异常。

## 实例属性和类属性

最后再来说说实例属性和类属性，前面我定义的类中用的属性都使用了self关键字作为前缀，
并在类的初始化函数中初始化这些属性，这些属于实例属性。类属性是直接定义在类中的属性。比如

```python
class Phone(object):
    weight = 20  # 类属性

    def __init__(self, brand, color):
        self.__brand = brand
        self.__color = color
```

当我们定义了一个类属性后，这个属性虽然归类所有，但类的所有实例都可以访问到。
所以一般我们会将一些公共常量定义成类属性。

下面两个都会打印20，说明类和实例都能访问这个类属性。

```python
print(SmartPhone.weight)
print(p.weight)
```
