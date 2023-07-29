---
title: 每天5分钟玩转Python（15） - 闭包
date: '2019-06-15 21:33:27 +0800'
toc: true
categories: [ python ]
tags: [ python教程 ]
---

闭包跟函数式紧密联系在一起的，介绍闭包之前先了解一下python中函数的高阶用法。比如嵌套函数、变量作用域等等。
<!-- more -->

## 变量作用域

变量作用域是程序运行时变量可被访问的范围，定义在函数内的变量是局部变量，
局部变量的作用范围只能是函数内部范围内，它不能在函数外引用。

```python
def test():
    num = 10 # 局部变量
print(num)  # NameError: name 'num' is not defined
```

定义在模块最外层的变量是全局变量，它是全局范围内可见的，当然在函数里面也可以读取到全局变量的。例如：

```python
num = 10 # 全局变量
def test():
    print(num)  # 10
```

再来一个示例

```python
outer = 6
def test(arg):
    print(arg)
    print(outer)  # 当函数执行到这一步会报错
                  # UnboundLocalError: local variable 'outer' referenced before assignment
    outer = 9     # 注意这里有了一行对于变量outer的赋值
test(1)
```

执行这个示例会报错，Python在编译函数时，发现对 outer 有赋值的操作，它判定 outer 是一个局部变量，
所以在打印 outer 时，它会去查询局部变量outer，发现并没有赋值，所以会抛出异常。

Python 就是这样设计的，它认为如果在函数体中对变量有赋值操作，则证明这个变量是一个局部变量，
并且它只会从局部变量中去读取数据。这样设计可以避免对全局变量的误操作。

可以使用 `global` 关键字来解决这个问题：

```python
outer = 6
def test(arg):
    print(arg)
    global outer
    print(outer)  # 6
    outer = 9 
    print(outer)  # 9
test(1)
```

## 嵌套函数

函数不仅可以定义在模块的最外层，还可以定义在另外一个函数的内部，
像这种定义在函数里面的函数称之为嵌套函数（nested function）例如：

```python
def print_msg():
    # print_msg 是外围函数
    msg = "hello python"
    def printer():
        # printer是嵌套函数
        print(msg)
    printer()
# 输出 hello python
print_msg()
```

## 闭包

当我们在函数内定义一个函数时，如果这个内部函数使用了外部函数的临时变量，
且外部函数的返回值是内部函数的引用时，那么返回的这个内部函数引用就是一个闭包。

有点绕口，直接上代码比较直观：

```python
def simple_avg():
    scores = []  #外部临时变量

    def inner_avg(val):  # 内部函数，用于计算平均值
        scores.append(val)  # 使用外部函数的临时变量
        return sum(scores) / len(scores)  # 返回计算出的平均值

    return inner_avg  # 外部函数返回内部函数引用


avg = simple_avg()
print(avg(10))  # 10
print(avg(11))  # 10.5
```

## nonlocal 关键字

上面的代码有一个小缺陷，就是有很多重复的计算，当我们传入一个新的值想要得到新的平均值时，
总是先将列表中所有值都计算求和，再求平均值，其实已经求和过的结果可以保存起来，
下一次只需要加最新的那个数就可以了。一般我们会想到这样修改：

```python
def simple_avg():
    scores = 0  # 将外部临时变量由 list 改为一个 整型数值
    count = 0   # 同时新增一个变量，记录个数

    def inner_avg(val):  # 内部函数，用于计算平均值
        scores += val  # 使用外部函数的临时变量
        count += 1
        return scores / count  # 返回计算出的平均值

    return inner_avg  # 外部函数返回内部函数引用

avg = simple_avg()
print(avg(10))  # 10
print(avg(11))  # 10.5
```

运行后报错：

```
UnboundLocalError: local variable 'scores' referenced before assignment
```

这里报错的原因是因为 `scores += val`，有了赋值操作，则认为 `scores` 是局部变量了。
而我们也没办法使用 `global` 关键字，因为此时 `score`s 和 `count` 是定义在 `get_avg` 函数内的，
它们俩也是一个局部变量。而为什么我们使用 list 时，没有出现这个问题呢？也是很好理解的，
因为我们使用的是 `list.append()` 方法，它没有赋值操作。

在 Python 3 中引入了一个关键词 nonlocal 解决了这一个问题：

```python
def simple_avg():
    scores = 0  # 将外部临时变量由 list 改为一个 整型数值
    count = 0   # 同时新增一个变量，记录个数

    def inner_avg(val):  # 内部函数，用于计算平均值
        nonlocal scores, count
        scores += val  # 使用外部函数的临时变量
        count += 1
        return scores / count  # 返回计算出的平均值

    return inner_avg  # 外部函数返回内部函数引用

avg = simple_avg()
print(avg(10))  # 10
print(avg(11))  # 10.5
```

结果正确返回。

## 闭包的应用

应用最多的就是装饰器，这个会在后面专门讲装饰器。

另外还有一个比较常用的应用就是惰性求值，具体的可以看下Django里面QuerySet的实现源码。

