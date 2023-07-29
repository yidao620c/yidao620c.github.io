---
title: 每天5分钟玩转Python（16） - 装饰器
date: '2019-06-16 09:58:33 +0800'
toc: true
categories: [ python ]
tags: [ python教程 ]
---

Python有着强大的表达式语法和函数特性，其中一个我的最爱便是装饰器。
在设计模式中，装饰器能够在不使用子类的情况下动态的修改函数、方法或类的功能。

当你需要扩展某个函数的功能却不想直接修改这个函数的时候，装饰器就可以派上用场了。
实现装饰器模式有很多种方法，但是Python通过强大的语法支持来让这个变得相当容易。

本质上来讲，装饰器是以包装器形式工作的，其实就是在执行目标函数之前或之后加入自己的逻辑，
而不需要改变目标函数本身就可以增强它的功能，也就是说装饰了它。

在这篇文章中我将深入讲解Python的函数装饰器，并通过一系列的源码示例来彻底讲清楚这个东西。

上一篇讲过闭包的概念，它的一个应用就是构造装饰器。
<!-- more -->

## 构造装饰器

函数装饰器就是已存在函数的一个包装器。

下面例子中我们先构造一个函数来用p标签包装其他函数返回的一个字符串。

```python
def get_text(name):
    return "hello, {0}".format(name)

def p_decorate(func):
    def func_wrapper(name):
        return "<p>{0}</p>".format(func(name))
    return func_wrapper

my_get_text = p_decorate(get_text)
print(my_get_text("zhangsan"))
```

打印结果为：

```
<p>hello, zhangsan</p>
```

这是我们的第一个装饰器，一个增强其他函数功能并返回新函数的函数。
为了让`get_text`函数被`p_decorate`装饰，我们只需要将`get_text`作为参数传给后者，
并将结果赋值给一个变量，然后就可以对这个变量函数调用就能实现效果了。

原来的函数有一个name参数，那么我们调用的时候将这个参数传递给装饰器函数就行了。

## 装饰器语法

Python通过一些语法糖让创建和使用装饰器变得相当简单。
我们并不需要使用语句`get_text = p_decorator(get_text)`来装饰get_text。
有一个快捷方式可以做到，它会在被装饰函数前面加一层装饰函数。装饰器的名字需要使用@前缀。

```python
def p_decorate(func):
   def func_wrapper(name):
       return "<p>{0}</p>".format(func(name))
   return func_wrapper

@p_decorate
def get_text(name):
   return "hello, {0}".format(name)

print(get_text("zhangsan"))
```

现在我们再考虑下利用2个其他的函数来装饰我们的`get_text`函数，在其输出结果上添加一个`div`和`strong`标签。

```python
def p_decorate(func):
   def func_wrapper(name):
       return "<p>{0}</p>".format(func(name))
   return func_wrapper

def strong_decorate(func):
    def func_wrapper(name):
        return "<strong>{0}</strong>".format(func(name))
    return func_wrapper

def div_decorate(func):
    def func_wrapper(name):
        return "<div>{0}</div>".format(func(name))
    return func_wrapper
```

如果我们使用原来的语法，那么就得这么写：

```python
get_text = div_decorate(p_decorate(strong_decorate(get_text)))
```

但是在python中，你就可以这样来定义了：

```python
@div_decorate
@p_decorate
@strong_decorate
def get_text(name):
   return "hello, {0}".format(name)

print(get_text("zhangsan"))
```

输出结果如下：

```
<div><p><strong>hello, zhangsan</strong></p></div>
```

上面需要注意的是装饰器的顺序，如果顺序不同，输出结果也会不一样。

## 装饰方法

在python中，其实方法就是第一个参数为当前对象的引用的函数而已，这方面的内容会在后面的面向对象编程讲到。
我们同样能够给方法构造装饰器，只需要将`self`参数放到包装函数中。

```python
def p_decorate(func):
   def func_wrapper(self):
       return "<p>{0}</p>".format(func(self))
   return func_wrapper

class Person(object):
    def __init__(self):
        self.name = "Neng"
        self.family = "Xiong"

    @p_decorate
    def get_fullname(self):
        return self.name+" "+self.family

my_person = Person()
print(my_person.get_fullname())
```

一个更好的做法是改造我们的装饰器使他们可以作用于函数以及类方法。
可以将*args和**kwargs作为包装器的参数，然后它就能接受任意数量的位置参数和关键字参数了。

```python
def p_decorate(func):
   def func_wrapper(*args, **kwargs):
       return "<p>{0}</p>".format(func(*args, **kwargs))
   return func_wrapper

class Person(object):
    def __init__(self):
        self.name = "Neng"
        self.family = "Xiong"

    @p_decorate
    def get_fullname(self):
        return self.name+" "+self.family

my_person = Person()
print(my_person.get_fullname())
```

## 给装饰器传递参数

回顾下上面的例子，你会发现例子中的装饰器太过冗余了。
3个装饰器(div_decorate,p_decorate, strong_decorate)拥有相同功能，只是使用了不同的标签包装而已。

我们可以做得更好，为什么不使用一种更加通用的方式，将标签作为参数传递进来呢？

```python
def tags(tag_name):
    def tags_decorator(func):
        def func_wrapper(*args, **kargs):
            return "<{0}>{1}</{0}>".format(tag_name, func(*args, **kargs))
        return func_wrapper
    return tags_decorator

@tags("div")
@tags("p")
@tags("strong")
def get_text(name):
    return "hello, "+name

print(get_text("zhangsan"))
```

## 调试被装饰函数

最后当我们调试被装饰函数时会发现它的名字、模块和文档字符串都发生了改变。

```python
print(get_text.__name__)
```

输出为：

```
func_wrapper
```

我们期望的输出应该是`get_text`，`get_text`的`__name__`、`__doc__` 和 `__module__`已经被包装函数覆盖了。

## 使用functools来解决

幸运的是python2.5版本以上有了一个functools包可以来解决这个问题。只需要简单在包装函数上标注`@wrap`标签即可。

```python
from functools import wraps

def tags(tag_name):
    def tags_decorator(func):
        @wraps(func)
        def func_wrapper(*args, **kargs):
            return "<{0}>{1}</{0}>".format(tag_name, func(*args, **kargs))
        return func_wrapper
    return tags_decorator

@tags("div")
@tags("p")
@tags("strong")
def get_text(name):
    return "hello, "+name

if __name__ == '__main__':
    print(get_text.__name__)  # get_text
    print(get_text.__doc__)  # returns some text
    print(get_text.__module__)  # __main__
    print(get_text('zhangsan'))
```

从结果可以看出`get_text`函数的属性都恢复正常了。

## 哪里使用装饰器

这篇文章中的例子相对来讲是比较简单的。它能给你的程序带来很大的方便。
一般来讲，装饰器用在需要扩展某个函数行为而又不想改变这个函数本身内容的时候。

我建议你查阅一下Python Decorator库来获取更多非常有用的装饰器。

## 更多阅读资源

下面是一个值得去查看的关于装饰器的其他资源列表：

* [什么是装饰器?](https://wiki.python.org/moin/PythonDecorators#What_is_a_Decorator)
* [Decorators I: Python装饰器入门](http://www.artima.com/weblogs/viewpost.jsp?thread=240808)
* [Python Decorators II: 装饰器参数](http://www.artima.com/weblogs/viewpost.jsp?thread=240845)
* [Python Decorators III: 一个基于装饰器的构建系统](http://www.artima.com/weblogs/viewpost.jsp?thread=241209)
* [Python装饰器指南 Matt Harrison](http://www.amazon.com/gp/product/B006ZHJSIM/ref=as_li_ss_tl?ie=UTF8&camp=1789&creative=390957&creativeASIN=B006ZHJSIM&linkCode=as2&tag=thcosh00-20)

