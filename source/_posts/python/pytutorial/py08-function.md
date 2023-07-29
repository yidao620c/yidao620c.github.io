---
title: 每天5分钟玩转Python（08） - 函数
date: '2019-06-08 22:13:33 +0800'
toc: true
categories: [ python ]
tags: [ python教程 ]
---

函数是组织好的、可重复使用的、用来实现单一或相关联功能的代码段。
函数能提高应用的模块性，和代码的重复利用率。你已经知道Python提供了许多内建函数，
比如`print()`。但你也可以自己创建函数，这被叫做用户自定义函数。
<!-- more -->

## 定义函数

在python中定义函数使用`def`开头，紧接着就是函数名称和圆括号`()`，在括号可以增加参数。
函数第一行语句可使用三引号格式的文档字符串来定义函数的说明。使用return语句定义函数返回值。

```python
def hello(name):
    """
    hello world
    """
    print('hello world, ' + name)
    return name
```

注意如果没有调用return语句，那么默认运行到最后会返回`None`

如果你还没想好函数的实现，可以再函数体内写个`pass`来定义一个空函数

```python
def nothing():
    pass
```

关键字`pass`是一个用来占位置的东西，很多时候语法要求一些地方必须有些语句才能编译通过。
但是你还没想好这里写啥，那就可以写个pass。比如下面的：

```python
if age < 15:
    pass
```

## 返回多个值

python中的函数一个很灵活的地方是可以返回多个值。实际上返回的是一个元组（tuple），看着是返回了多个值。

```python
def multi_result(a, b, c):
    return a + 1, b + 1, c + 1

r = multi_result(1, 2, 3)
print(r)
```

## 参数

python中的函数定义非常灵活，最重要是因为参数的定义上。除了必选的位置参数外，还有关键字参数、默认参数、可变参数。
满足多种场景的应用。

### 位置参数

这是最常见的参数，比如定义一个求差的函数如下

```python
def my_subtract(x, y):
    return x - y
```

调用这个函数的时候，必选传入x和y的值，并且顺序不能变。

```python
print(my_subtract(5, 3))  # 打印2
```

### 默认参数

如果想让某个参数有个默认值，如果调用的时候没有传这个参数就使用默认值。则可以使用如下格式定义这个函数

```python
def my_subtract(x, y=0):
    return x - y
```

其中参数y有个默认值为0。这时候如果你调用的时候传入了y的值就使用真实的y值，如果没有传y就使用默认值0

```python
print(my_subtract(5, 3))  # 打印2
print(my_subtract(5))  # 打印5
```

对于默认参数，有个约束就是必选的位置参数在前面，默认参数在后面。
还有一个强烈建议就是，默认参数所指的对象一定要是一个不可变对象。

### 可变参数

可变参数的意思就是传入的参数个数是可以变化的。python中使用`func(*args)`来定义可变参数。
比如如果要实现对于多个数的相加运算，你可能首先想到的是将这多个数放到一个列表中，然后将这个列表当做参数传给某个函数。

```python
def calc(numbers):
    sum = 0
    for n in numbers:
        sum = sum + n * n
    return sum
```

调用的时候，先将多个数拼装成一个list或tuple，然后再调用这个函数：

```python
calc([1, 2, 3])
```

其实我们可以将这个numbers参数变成可变参数，只需要在这个参数前面加上`*`即可。

```python
def calc(*numbers):
    sum = 0
    for n in numbers:
        sum = sum + n * n
    return sum
```

然后你调用的时候就简单了

```python
calc(1, 2, 3)
```

但如果你已经有了一个list对象，也想调用这个方法，咋办呢？Python允许你在list或tuple前面加一个`*`号，
把list或tuple的元素变成可变参数传进去：

```python
nums = [1, 2, 3]
calc(*nums)
14
```

`*nums`表示把`nums`这个list的所有元素作为可变参数传进去。这种写法相当有用，而且很常见。

### 关键字参数

python中通过在变量的前面加`**`将参数变成关键字参数。关键字参数允许你传入含参数名的参数，
这些关键字参数在函数内部自动组装为一个dict，比如：

```python
def person(name, age, **kw):
    print('name:', name, 'age:', age, 'other:', kw)
```

在这个函数里，前面两个参数是必选的位置参数。后面的`kw`是关键字参数。传入的时候使用`name=value`的形式。比如

```python
person('XiongNeng', 24, city='XiAn', job='Engineer')
```

实际调用中，`kw`变成了dict对象，值为`{'city': 'XiAn', 'job': 'Engineer'}`

如果已经有了一个dict对象，然后又想去调用这个方法咋整。类似可变参数，
可以再这个dict对象前面加`**`作为关键字参数传入进去即可。

```python
param = {'city': 'XiAn', 'job': 'Engineer'}
person('name', 24, **param)
```

输出：

```
name: name age: 24 other: {'city': 'XiAn', 'job': 'Engineer'}
```

### 命名关键字参数

如果你还想控制调用方传入的关键字参数名，可使用如下语法：

```python
def person(name, age, *, city, job):
    print(name, age, city, job)
```

和关键字参数`**kw`不同，命名关键字参数需要一个特殊分隔符`*`，`*`后面的参数被视为命名关键字参数。

调用方式跟上面一样，只不过这次你传入的关键字参数必行是`city`和`job`。如果是其他的就会报错

```python
person('XiongNeng', 24, city='XiAn', job='Engineer')
```

如果函数定义中已经有了一个可变参数，后面跟着的命名关键字参数就不再需要一个特殊分隔符*了：

```python
def person(name, age, *args, city, job):
    print(name, age, args, city, job)
```

命名关键字参数必须传入参数名和参数值，不过你可以给命名关键字参数加上默认值，那这时候就不是必须传了。

```python
def person(name, age, *, city='Beijing', job):
    print(name, age, city, job)
```

调用的时候，不需要传入city参数

```python
person('XiongNeng', 24, job='Engineer')
```

### 参数组合

在Python中定义函数，可以用必选参数、默认参数、可变参数、关键字参数和命名关键字参数，这5种参数都可以组合使用。
但是请注意，参数定义的顺序必须是：必选参数、默认参数、可变参数、命名关键字参数和关键字参数。

```python
def f1(a, b, c=0, *args, **kw):
    print('a =', a, 'b =', b, 'c =', c, 'args =', args, 'kw =', kw)

def f2(a, b, c=0, *, d, **kw):
    print('a =', a, 'b =', b, 'c =', c, 'd =', d, 'kw =', kw)
```

在函数调用的时候，Python解释器自动按照参数位置和参数名把对应的参数传进去。

对于任意函数，都可以通过类似`func(*args, **kw)`的形式调用它，无论它的参数是如何定义的。

注意：使用`*args`和`**kw`是Python的习惯写法，当然也可以用其他参数名，但最好使用习惯用法。

函数的基础部分讲解完了。Python中函数式一等公民，所以后续还会讲解对于更高阶的使用方式。

