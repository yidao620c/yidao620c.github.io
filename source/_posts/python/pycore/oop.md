---
title: python核心 - 面向对象编程
date: 2015-12-06 22:22:22 +0800
toc: true
categories: [ python ]
tags: [ python核心 ]
---

面向对象编程——Object Oriented Programming，简称OOP，是一种程序设计思想。
OOP把对象作为程序的基本单元，一个对象包含了数据和操作数据的函数。

面向过程的程序设计把计算机程序视为一系列的命令集合，即一组函数的顺序执行。
为了简化程序设计，面向过程把函数继续切分为子函数，即把大块函数通过切割成小块函数来降低系统的复杂度。

而面向对象的程序设计把计算机程序视为一组对象的集合，而每个对象都可以接收其他对象发过来的消息，
并处理这些消息，计算机程序的执行就是一系列消息在各个对象之间传递。

在Python中，所有数据类型都可以视为对象，当然也可以自定义对象。
自定义的对象数据类型就是面向对象中的类（Class）的概念。
<!-- more -->

给对象发消息实际上就是调用对象对应的关联函数(或者叫绑定函数)，我们称之为对象的方法（Method）。

面向对象的设计思想是从自然界中来的，因为在自然界中，类（Class）和实例（Instance）的概念是很自然的。
Class是一种抽象概念，比如我们定义一个叫Person的Class指定人类这个概念，
而实例（Instance）则是一个个具体的Person，比如，张三和李四是两个具体的Person。

所以，面向对象的设计思想是抽象出Class，根据Class创建Instance。
之前面向过程或函数式编程时我们通常说程序设计等价于算法+数据结构。
算法和数据是分开的，而面向对象的抽象程度又比函数要高，因为一个Class既包含数据，又包含操作数据的方法。

## 类的定义

最基本的方式

```python
class Student(object):

    def __init__(self, name, score):
        self.name = name
        self.score = score

    def print_score(self):
        print('%s: %s' % (self.name, self.score))


zsan = Student('Zhang Shan', 22)
zsan.print_score()
```

第一个__init__是实例初始化会调用的方法，第二个自定义方法。
所有方法第一个参数必须是self，调用方法时这个参数不需要传递。

和静态语言不同，Python允许对实例变量绑定任何数据，也就是说，对于两个实例变量，
虽然它们都是同一个类的不同实例，但拥有的变量名称都可能不同。你在实例化后可以给他再增加属性。

## 单下划线_和双下划线__

在模块中：

1. 单下划线_表示这个属性和函数是私有的，不应该直接访问
2. 双下划线没有什么意义
3. 对于from module import *这样的导入，不管单下划线还是双下划线都不会被导入
4. 对于import module这样的导入都可以使用，但不建议使用，PEP8会告警调用单下划线_

在类中：

1. 单下划线_表示方法是私有方法或属性，不建议直接访问，PEP8会告警
2. 双下划线__表示final方法或属性防止被子类覆盖，python会自己更改它的名称为_classname__name，不能直接访问到，可用来定义私有属性和方法
3. 最佳方案是私有属性用双下划线__，而私有方法用单下划线_

前后双下划线比如__init__是python内部特殊的名字，自己没事就别定义这样的东西了

总的来说就是，Python本身没有任何机制阻止你干坏事，一切全靠约定和自觉。

## 私有属性

在Python中，实例的变量名如果以双下划线__开头，就变成了一个私有变量（private），只有内部可以访问，外部不能访问。

```python
class Student(object):
    def __init__(self, name, score):
        self.__name = name
        self.__score = score

    def print_score(self):
        print('%s: %s' % (self.__name, self.__score))
```

这样就确保了外部代码不能随意修改对象内部的状态，这样通过访问限制的保护，代码更加健壮。

## 继承和多态

基本用法

```python
class Animal(object):
    def run(self):
        print('Animal is running...')


class Dog(Animal):
    def run(self):
        print('Dog is running...')


class Cat(Animal):
    def run(self):
        print('Cat is running...')


dog = Dog()
dog.run()

cat = Cat()
cat.run()
```

获取对象的信息：

```python
# 判断一个变量是否是某个类型可以用isinstance()
isinstance(cat, Animal)

# 获得一个对象的所有属性和方法，可以使用dir()函数，
# 它返回一个包含字符串的list，比如，获得一个str对象的所有属性和方法
dir('ABC')

# 配合getattr()、setattr()以及hasattr()，我们可以直接操作一个对象的状态
hasattr(obj, 'x')  # 有属性'x'吗？
setattr(obj, 'y', 19)  # 设置一个属性'y'
getattr(obj, 'y')  # 获取属性'y'
getattr(obj, 'y'， 'none')  # 获取属性'y'，带默认值
```

pythony允许多继承，也就是我们通常所说的MixIn。MixIn的目的就是给一个类增加多个功能，在设计类的时候，
我们优先考虑通过多重继承来组合多个MixIn的功能，而不是设计多层次的复杂的继承关系。

比如，编写一个多进程模式的TCP服务，定义如下：

```python
class MyTCPServer(TCPServer, ForkingMixIn):
    pass
```

编写一个多线程模式的UDP服务，定义如下：

```python
class MyUDPServer(UDPServer, ThreadingMixIn):
    pass
```

这样一来，我们不需要复杂而庞大的继承链，只要选择组合不同的类的功能，就可以快速构造出所需的子类

## 使用__slots__

给一个实例绑定一个新的方法：

```python
def set_age(self, age):  # 定义一个函数作为实例方法
    self.age = age


from types import MethodType

s.set_age = MethodType(set_age, s)  # 给实例绑定一个方法
s.set_age(25)  # 调用实例方法
s.age  # 测试结果
25
```

如果我们想要限制实例的属性怎么办？
Python允许在定义class的时候，定义一个特殊的__slots__变量，来限制该class实例能添加的属性

```python
class Student(object):
    __slots__ = ('name', 'age')  # 用tuple定义允许绑定的属性名称
```

注：__slots__定义的属性仅对当前类实例起作用，对继承的子类是不起作用的

## 使用@property

python最佳编程实践推荐我们不要像java那样去调用getter和setter，而是使用装饰器@property

Python内置的@property装饰器就是负责把一个方法变成属性调用

把一个getter方法变成属性，只需要加上@property就可以了，
此时，@property本身又创建了另一个装饰器@score.setter，负责把一个setter方法变成属性赋值

```python
class Student(object):
    @property
    def score(self):
        return self._score

    @score.setter
    def score(self, value):
        if not isinstance(value, int):
            raise ValueError('score must be an integer!')
        if value < 0 or value > 100:
            raise ValueError('score must between 0 ~ 100!')
        self._score = value


s = Student()
s.score = 60
s.score
```

还可以定义只读属性，只定义getter方法，不定义setter方法就是一个只读属性

## 枚举类

通常我们需要用到常量，并且是一些有意义的常量。我们可以通过枚举类Enum实现

```python
from enum import Enum, unique


@unique
class Weekday(Enum):
    Sun = 0  # Sun的value被设定为0
    Mon = 1
    Tue = 2
    Wed = 3
    Thu = 4
    Fri = 5
    Sat = 6


day1 = Weekday.Mon
print(day1 == Weekday.Mon)
print(Weekday.Tue)
print(Weekday['Tue'])
print(Weekday.Tue.value)
```

## 使用元类

这个已经在"python核心-元类"里面讲解很详细，这里省略...
