---
title: python核心 - 装饰器
date: 2015-12-03 10:06:22 +0800
toc: true
categories: [ python ]
tags: [ python核心 ]
---

python有着强大的表达式语法和函数特性，其中一个我的最爱便是装饰器。
在设计模式中，装饰器能够在不使用子类的情况下动态的修改函数、方法或类的功能。

当你需要扩展某个函数的功能却不想直接修改这个函数的时候，装饰器就可以派上用场了。
实现装饰器模式有很多种方法，但是python通过强大的语法支持来让这个变得相当容易。

在这篇文章中我将深入讲解Python的函数装饰器，并通过一系列的源码示例来彻底讲清楚这个东西。
所有例子都在Python2.7下运行通过，不过只需要稍作改变就可以运行在Python3上了，
甚至我猜测什么都不用改变都可以的，读者可以自己去试试。

本质上来讲，装饰器是以包装器形式工作的，其实就是在执行目标函数之前或之后加入自己的逻辑，
而不需要改变目标函数本身就可以增强它的功能，也就是说装饰了它。
<!-- more -->

### 你需要知道的函数

在深入讨论之前，有一些基本的概念需要讲明清楚。
在Python中，函数是一等公民，它们就是对象，因此我们可以使用它来做很多事。

1.把函数赋值给某个变量：

```python
def greet(name):
    return "hello " + name


greet_someone = greet
print
greet_someone("John")

# Outputs: hello John
```

2.在某个函数内部定义另外一个函数：

```python
def greet(name):
    def get_message():
        return "Hello "

    result = get_message() + name
    return result


print
greet("John")

# Outputs: Hello John
```

3.函数可以被当做参数传递给另外一个函数：

```python
def greet(name):
    return "Hello " + name


def call_func(func):
    other_name = "John"
    return func(other_name)


print
call_func(greet)

# Outputs: Hello John
```

4.函数返回值可以是其他函数：

```python
def compose_greet_func():
    def get_message():
        return "Hello there!"

    return get_message


greet = compose_greet_func()
print
greet()

# Outputs: Hello there!
```

5.内部函数可以访问包含它的函数的局部变量：

其实就是我们所说的闭包，在构建装饰器的时候这是一个非常有用的模式。另外还要注意，Python只允许读取外部变量而不允许修改。

观察一下下面的代码，注意我们是如何通过修改上面实例代码来读取外部函数中的name参数值并返回一个新的函数的。

```python
def compose_greet_func(name):
    def get_message():
        return "Hello there " + name + "!"

    return get_message


greet = compose_greet_func("John")
print
greet()

# Outputs: Hello there John!
```

### Python中的闭包

既然讲到这里了，就不得不去提一下python中的闭包了。

定义：如果在一个内部函数里，对在外部作用域（外部函数的局部变量或者参数，但不是在全局作用域）的变量进行引用，那么内部函数就被认为是闭包（closure）

分解来说，包含下面3个条件：

1. 需要函数嵌套, 就是一个函数里面再写一个函数.
2. 外部函数需要返回一个内部函数的引用
3. 外部函数中有一些局部变量或者参数, 并且, 这些局部变量或参数在内部函数中有使用

一些概念:

1. 自由变量: 外部函数中定义的局部变量, 并且在内部函数中被使用
2. 闭包: 那个使用了自由变量并被返回的内部函数就称为闭包

支持闭包的语言有这样的特性：

1. 函数是一阶值（First-class value），即函数可以作为另一个函数的返回值或参数，还可以作为一个变量的值
2. 函数可以嵌套定义，即在一个函数内部可以定义另一个函数

代码示例：

```python
def line_conf():
    b = 15

    def line(x):
        return 2 * x + b

    return line  # return a function object


b = 5
my_line = line_conf()  # 返回一个闭包函数
print(my_line(5))
```

### 构造装饰器

函数装饰器就是已存在函数的一个包装器。我们把上面的这些结合起来就能构建一个装饰器了。

下面例子中我们先构造一个函数来用p标签包装其他函数返回的一个字符串。

```python
def get_text(name):
    return "lorem ipsum, {0} dolor sit amet".format(name)


def p_decorate(func):
    def func_wrapper(name):
        return "<p>{0}</p>".format(func(name))

    return func_wrapper


my_get_text = p_decorate(get_text)

print
my_get_text("John")

# <p>Outputs lorem ipsum, John dolor sit amet</p>
```

这是我们的第一个装饰器——一个增强其他函数功能并返回新函数的函数。
为了让get_text函数被p_decorate装饰，我们只需要将get_text作为参数传给后者，
并将结果赋值给一个变量，然后就可以对这个变量函数调用就能实现效果了。

```python
get_text = p_decorate(get_text)

print
get_text("John")

# Outputs lorem ipsum, John dolor sit amet
```

主要原来的函数有一个name参数，那么我们调用的时候将这个参数传递给装饰器函数就行了。

### Python的装饰器语法

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
    return "lorem ipsum, {0} dolor sit amet".format(name)


print
get_text("John")

# Outputs <p>lorem ipsum, John dolor sit amet</p>
```

现在我们再考虑下利用2个其他的函数来装饰我们的get_text函数，在其输出结果上添加一个div和strong标签。

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
    return "lorem ipsum, {0} dolor sit amet".format(name)


print
get_text("John")

# Outputs <div><p><strong>lorem ipsum, John dolor sit amet</strong></p></div>
```

上面需要注意的是装饰器的顺序，如果顺序不同，输出结果也会不一样。

### 装饰方法

在python中，其实方法就是第一个参数为当前对象的引用的函数而已。
我们同样能够给方法构造装饰器，只需要将self参数放到包装函数中。

```python
def p_decorate(func):
    def func_wrapper(self):
        return "<p>{0}</p>".format(func(self))

    return func_wrapper


class Person(object):
    def __init__(self):
        self.name = "John"
        self.family = "Doe"

    @p_decorate
    def get_fullname(self):
        return self.name + " " + self.family


my_person = Person()
print
my_person.get_fullname()
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
        self.name = "John"
        self.family = "Doe"

    @p_decorate
    def get_fullname(self):
        return self.name + " " + self.family


my_person = Person()

print
my_person.get_fullname()
```

### 给装饰器传递参数

回顾下上面的例子，你会发现例子中的装饰器太过冗余了。
3个装饰器(div_decorate,p_decorate, strong_decorate)拥有相同功能，只是使用了不同的标签包装而已。

我们可以做得更好，为什么不使用一种更加通用的方式，将标签作为参数传递进来呢？

```python
def tags(tag_name):
    def tags_decorator(func):
        def func_wrapper(name):
            return "<{0}>{1}</{0}>".format(tag_name, func(name))

        return func_wrapper

    return tags_decorator


@tags("p")
def get_text(name):
    return "Hello " + name


print
get_text("John")

# Outputs <p>Hello John</p>
```

### 调试被装饰函数

最后当我们调试被装饰函数时会发现它的名字、模块和文档字符串都发生了改变。

```python
print
get_text.__name__
# Outputs func_wrapper
```

我们期望的输出应该是get_text，get_text的__name__、__doc__ 和 __module__已经被包装函数覆盖了。

### 使用functools来解决

幸运的是python2.5版本以上有了一个functools包可以来解决这个问题。
只需要简单在包装函数上标注@wrap标签即可。

```python
from functools import wraps


def tags(tag_name):
    def tags_decorator(func):
        @wraps(func)
        def func_wrapper(*args, **kargs):
            return "<{0}>{1}</{0}>".format(tag_name, func(*args, **kargs))

        return func_wrapper

    return tags_decorator


def pptags(func):
    @wraps(func)
    def func_wrapper(*args, **kargs):
        return "<{0}>{1}</{0}>".format('pp', func(*args, **kargs))

    return func_wrapper


@tags('p')
def get_text(name):
    """returns some text with p"""
    return "Hello " + name


@pptags
def get_text_pp(name):
    """returns some text with pp"""
    return "Hello " + name


if __name__ == '__main__':
    print(get_text.__name__)  # get_text
    print(get_text.__doc__)  # returns some text
    print(get_text.__module__)  # __main__
    print(get_text('韩梅梅'))

    print(get_text_pp.__name__)  # get_text
    print(get_text_pp.__doc__)  # returns some text
    print(get_text_pp.__module__)  # __main__
    print(get_text_pp('李雷'))
```

从结果可以看出get_text函数的属性都恢复正常了。

### 哪里使用装饰器

这篇文章中的例子相对来讲是比较简单的。它能给你的程序带来很大的方便。
一般来讲，装饰器用在需要扩展某个函数行为而又不想改变这个函数本身内容的时候。

我建议你查阅一下Python Decorator库来获取更多非常有用的装饰器。

### 更多阅读资源

下面是一个值得去查看的关于装饰器的其他资源列表：

* [什么是装饰器?](https://wiki.python.org/moin/PythonDecorators#What_is_a_Decorator)
* [Decorators I: Python装饰器入门](http://www.artima.com/weblogs/viewpost.jsp?thread=240808)
* [Python Decorators II: 装饰器参数](http://www.artima.com/weblogs/viewpost.jsp?thread=240845)
* [Python Decorators III: 一个基于装饰器的构建系统](http://www.artima.com/weblogs/viewpost.jsp?thread=241209)
* [Python装饰器指南 Matt Harrison](http://www.amazon.com/gp/product/B006ZHJSIM/ref=as_li_ss_tl?ie=UTF8&camp=1789&creative=390957&creativeASIN=B006ZHJSIM&linkCode=as2&tag=thcosh00-20)

到此为止Python装饰器概率已经介绍完了。我希望你能从中受益，如果你有任何的建议或问题，可以在评论中提出来。祝您编程快乐！

