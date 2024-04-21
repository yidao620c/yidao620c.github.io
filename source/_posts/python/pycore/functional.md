---
title: python核心 - 函数式编程
date: 2015-10-22 10:06:22 +0800
toc: true
categories: [ python ]
tags: [ python核心 ]
---

函数式编程就是一种抽象程度很高的编程范式，纯粹的函数式编程语言编写的函数没有变量(或者说不能给变量重新赋值)，
因此，任意一个函数，只要输入是确定的，输出就是确定的，这种纯函数我们称之为没有副作用。而允许使用变量的程序设计语言，
由于函数内部的变量状态不确定，同样的输入，可能得到不同的输出，因此，这种函数是有副作用的。

函数式编程（请注意多了一个"式"字）——Functional Programming，虽然也可以归结到面向过程的程序设计，但其思想更接近数学计算。
它的一个特点就是，允许把函数本身作为参数传入另一个函数，还允许返回一个函数！

Python对函数式编程提供部分支持。由于Python允许使用变量，因此，Python不是纯函数式编程语言。
<!-- more -->

### python中的函数

先讲解一下python中函数的基础

#### 默认参数

默认参数很有用，但使用不当，也会掉坑里。默认参数有个最大的坑，演示如下：

先定义一个函数，传入一个list，添加一个END再返回：

```python
def add_end(L=[]):
    L.append('END')
    return L
```

Python函数在定义的时候，默认参数L的值就被计算出来了，即[]，因为默认参数L也是一个变量，它指向对象[]
，每次调用该函数，如果改变了L的内容，则下次调用时，默认参数的内容就变了，不再是函数定义时的[]了。

所以，定义默认参数要牢记一点：默认参数必须指向不变对象！

要修改上面的例子，我们可以用None这个不变对象来实现：

```python
def add_end(L=None):
    if L is None:
        L = []
    L.append('END')
    return L
```

#### 可变参数

在参数前面加一个*号即可变成可变参数

```python
def calc(*numbers):
    sum = 0
    for n in numbers:
        sum = sum + n * n
    return sum
```

如果已经有一个list或者tuple，要调用一个可变参数怎么办？Python允许你在list或tuple前面加一个*号，把list或tuple的元素变成可变参数传进去：

```python
>> > nums = [1, 2, 3]
>> > calc(*nums)
14
```

*nums表示把nums这个list的所有元素作为可变参数传进去。这种写法相当有用，而且很常见。在函数内部，如意又要调用其他的有可变参数的函数，可以这样做。

#### 关键字参数

可变参数允许你传入0个或任意个参数，这些可变参数在函数调用时自动组装为一个tuple。而关键字参数允许你传入0个或任意个含参数名的参数，这些关键字参数在函数内部自动组装为一个dict。请看示例：

```python
def person(name, age, **kw):
    print('name:', name, 'age:', age, 'other:', kw)

>> > person('Bob', 35, city='Beijing')
name: Bob
age: 35
other: {'city': 'Beijing'}
>> > person('Adam', 45, gender='M', job='Engineer')
name: Adam
age: 45
other: {'gender': 'M', 'job': 'Engineer'}
```

和可变参数类似，也可以先组装出一个dict，然后，把该dict转换为关键字参数传进去，在参数前面加**

```python
>> > extra = {'city': 'Beijing', 'job': 'Engineer'}
>> > person('Jack', 24, **extra)
name: Jack
age: 24
other: {'city': 'Beijing', 'job': 'Engineer'}
```

注：kw获得的dict是extra的一份拷贝，对kw的改动不会影响到函数外的extra。

#### 命名关键字参数

如果要限制关键字参数的名字，就可以用命名关键字参数，例如，只接收city和job作为关键字参数。这种方式定义的函数如下：

```python
def person(name, age, *, city, job):
    print(name, age, city, job)
```

和关键字参数**kw不同，命名关键字参数需要一个特殊分隔符*，*后面的参数被视为命名关键字参数。

调用方式如下：

```python
>> > person('Jack', 24, city='Beijing', job='Engineer')
Jack
24
Beijing
Engineer
```

如果函数定义中已经有了一个可变参数，后面跟着的命名关键字参数就不再需要一个特殊分隔符*了：

```python
def person(name, age, *args, city, job):
    print(name, age, args, city, job)
```

命名关键字参数必须传入参数名，这和位置参数不同。如果没有传入参数名，调用将报错

#### 参数组合

在Python中定义函数，可以用必选参数、默认参数、可变参数、关键字参数和命名关键字参数，这5种参数都可以组合使用。
但是请注意，参数定义的顺序必须是：必选参数、默认参数、可变参数、命名关键字参数和关键字参数。

对于任意函数，都可以通过类似func(*args, **kw)的形式调用它，无论它的参数是如何定义的。

### 高阶函数

把函数作为参数传入，这样的函数称为高阶函数，函数式编程就是指这种高度抽象的编程范式。

#### 内建map()函数

map()函数接收两个参数，一个是函数，一个是Iterable，map将传入的函数依次作用到序列的每个元素，并把结果作为新的Iterator返回。

举例说明，比如我们有一个函数f(x)=x2，要把这个函数作用在一个list [1, 2, 3, 4, 5, 6, 7, 8, 9]上，就可以用map()实现如下：

![](https://xnstatic-1253397658.file.myqcloud.com/py04f01.png)

```python
>> >

def f(x):
    ...


return x * x
...
>> > r = map(f, [1, 2, 3, 4, 5, 6, 7, 8, 9])
>> > list(r)
[1, 4, 9, 16, 25, 36, 49, 64, 81]
```

map()传入的第一个参数是f，即函数对象本身。由于结果r是一个Iterator，Iterator是惰性序列，因此通过list()
函数让它把整个序列都计算出来并返回一个list。

所以，map()作为高阶函数，事实上它把运算规则抽象了

#### reduce()函数

再看reduce的用法。reduce把一个函数作用在一个序列[x1, x2, x3, ...]上，这个函数必须接收两个参数，reduce把结果继续和序列的下一个元素做累积计算，其效果就是：

```python
reduce(f, [x1, x2, x3, x4]) = f(f(f(x1, x2), x3), x4)
```

比方说对一个序列求和，就可以用reduce实现：

```python
>> > from functools import reduce
>> >

def add(x, y):
    ...


return x + y
...
>> > reduce(add, [1, 3, 5, 7, 9])
25
```

把序列[1, 3, 5, 7, 9]变换成整数13579，reduce就可以派上用场：

```python
>> > from functools import reduce
>> >

def fn(x, y):
    ...


return x * 10 + y
...
>> > reduce(fn, [1, 3, 5, 7, 9])
13579
```

配合map，将str转换成int

```python
from functools import reduce


def str2int(s):
    def fn(x, y):
        return x * 10 + y

    def char2num(s):
        return {'0': 0, '1': 1, '2': 2, '3': 3, '4': 4, '5': 5, '6': 6, '7': 7, '8': 8, '9': 9}[s]

    return reduce(fn, map(char2num, s))
```

用lambda函数进一步简化成：

```python
from functools import reduce


def str2int(s):
    def char2num(s):
        return {'0': 0, '1': 1, '2': 2, '3': 3, '4': 4, '5': 5, '6': 6, '7': 7, '8': 8, '9': 9}[s]

    return reduce(lambda x, y: x * 10 + y, map(char2num, s))
```

#### 内建的filter()函数

和map()类似，filter()也接收一个函数和一个序列。和map()不同的是，filter()把传入的函数依次作用于每个元素，然后根据返回值是True还是False决定保留还是丢弃该元素。

例如，在一个list中，删掉偶数，只保留奇数，可以这么写：

```python
list(filter(lambda n: n % 2 == 1, [1, 2, 4, 5, 6, 9, 10, 15]))
```

注意到filter()函数返回的是一个Iterator，也就是一个惰性序列，所以要强迫filter()完成计算结果，需要用list()函数获得所有结果并返回list。

#### 埃氏筛法求素数

计算素数的一个方法是埃氏筛法，它的算法理解起来非常简单：

首先，列出从2开始的所有自然数，构造一个序列：

2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, ...

取序列的第一个数2，它一定是素数，然后用2把序列的2的倍数筛掉：

3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, ...

取新序列的第一个数3，它一定是素数，然后用3把序列的3的倍数筛掉：

5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, ...

取新序列的第一个数5，然后用5把序列的5的倍数筛掉：

7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, ...

不断筛下去，就可以得到所有的素数。

用Python来实现这个算法，可以先构造一个从3开始的奇数序列：

```python
def _odd_iter():
    n = 1
    while True:
        n += 2
        yield n
```

注意这是一个生成器，并且是一个无限序列。

然后定义一个筛选函数（返回一个闭包）：

```python
def _not_divisible(n):
    return lambda x: x % n > 0
```

最后，定义一个生成器，不断返回下一个素数：

```python
def primes():
    yield 2
    it = _odd_iter()  # 初始序列
    while True:
        n = next(it)  # 返回序列第一个值
        yield n
        it = filter(_not_divisible(n), it)  # 构造新序列
```

注意到Iterator是惰性计算的序列，所以我们可以用Python表示"全体自然数"，"全体素数"这样的序列，而代码非常简洁。

#### 内建的排序函数sorted()

sorted()函数也是一个高阶函数，它还可以接收一个key函数来实现自定义的排序，例如按绝对值大小排序：

```python
sorted([36, 5, -12, 9, -21], key=abs)
sorted(['bob', 'about', 'Zoo', 'Credit'], key=str.lower)
sorted(['bob', 'about', 'Zoo', 'Credit'], key=str.lower, reverse=True)  # 反向排序
```

### 偏函数

functools模块提供了一个工具让我们创建偏函数，简单来讲，偏函数就是把一个函数的某些参数给固定住（也就是设置默认值），返回一个新的函数，调用这个新函数会更简单。

```python
>> > import functools
>> > int2 = functools.partial(int, base=2)
>> > int2('1000000')
64
>> > int2('1010101')
85
```

创建偏函数时，实际上可以接收函数对象、*args和**kw这3个参数，当传入：

```python
int2 = functools.partial(int, base=2)
```

实际上固定了int()函数的关键字参数base

当传入：

```python
max2 = functools.partial(max, 10)
```

实际上会把10作为*args的一部分自动加到左边，也就是：

```python
max2(5, 6, 7)
```

相当于：

```python
args = (10, 5, 6, 7)
max(*args)
```
