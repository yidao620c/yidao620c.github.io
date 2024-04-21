---
title: 每天5分钟玩转Python（13） - 函数式编程之map/reduce
date: '2019-06-13 10:28:33 +0800'
toc: true
categories: [ python ]
tags: [ python教程 ]
---

函数式（Functional Programming）编程是一种抽象程度非常高的编程范式，纯粹的函数式编程中没有变量。
对于任意一个函数，输入确定则输出也确定，这种纯函数没有任何副作用，非常适合高并发场景。
比如Lisp语言就是一个纯粹的函数式编程语言。

另外函数式编程中，函数可作为参数传入另外一个函数，也可作为返回值，跟变量一样。
由于Python中存在变量，因此并不是纯函数式编程语言。
<!-- more -->

## map/reduce

这个东西最初出自于Google的论文《MapReduce: Simplified Data Processing on Large Clusters》。
在大数据领域很常见。

## map()函数

map()函数接受两个参数，一个是函数，一个是`Iterable`，map将这个函数作用于`Iterable`的每个元素上。
并且返回的仍然是一个`Iterable`。

比如我有一个函数f(x)=x^2，要把这个函数作用于一个列表`[1, 2, 3, 4, 5, 6, 7, 8, 9]`上。

![](https://xnstatic-1253397658.file.myqcloud.com/py13_001.png)

python中可以用map()函数实现：

```python
def f(x):
    return x * x

r = map(f, [1, 2, 3, 4, 5, 6, 7, 8, 9])
print(list(r))
```

## reduce()函数

reduce()也是接受两个参数，一个是函数，一个是`Iterable`。将函数作用在`Iterable`上，这个函数必须接受两个参数，
reduce会首先取列表最前面2个元素运算，然后将结果作为第一个参数，将`Iterable`中下一个元素作为第二个参数继续运算。
迭代一直持续到将`Iterable`元素耗尽为止。效果类似：

```
reduce(f, [x1, x2, x3, x4]) = f(f(f(x1, x2), x3), x4)
```

最常见的应用就是序列求和了，可以用reduce函数实现：

```python
from functools import reduce

def add(x, y):
    return x + y

result = reduce(add, [1, 3, 5, 7, 9])
print(result)
```

你可能觉得这个求和也太简单了，我再举个例子。将整数序列变成数字一起组合的整数。
比如将`[5, 4, 3, 2, 1]`转换成54321。可以用reduce函数实现：

```python
from functools import reduce

def fn(x, y):
    return x * 10 + y

print(reduce(fn, [5, 4, 3, 2, 1]))
```

再继续发散思维，由于字符串也是一个序列，如果再结合函数`int`就能把一个整数字符串转换成对应的整数。

```python
from functools import reduce

def str2int(s):
    return reduce(lambda x, y: x * 10 + y, map(int, s))
```

注意上面还使用到了lambda函数，正好趁这个机会回顾一下lambda函数的使用方法。

如果不使用`int`函数，也可以自己实现：

```python
from functools import reduce

def char2num(s):
    return {'0': 0, '1': 1, '2': 2, '3': 3, '4': 4, '5': 5, '6': 6, '7': 7, '8': 8, '9': 9}[s]

def str2int(s):
    return reduce(lambda x, y: x * 10 + y, map(char2num, s))
```

最后的效果都是一样的。

