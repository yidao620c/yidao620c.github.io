---
title: 每天5分钟玩转Python（04） - 基本语法
date: '2019-06-04 09:37:12 +0800'
toc: true
categories: [ python ]
tags: [ python教程 ]
---

学一门语言最开始还是得先了解一下这门语言的基本语法，Python跟C语言语法有很大差别。
并且由于是一门脚本语言，语法比较的简单。这篇系列讲的都是Python3，所以语法也最新的3.x的语法。
<!-- more -->

## 源文件

Python的源文件一般以.py结尾，同时默认是以UTF-8编码。对于每个python源文件，在最开始声明如下两行是一个很好地习惯

```python
#!/usr/bin/env python
# -*- encoding: utf-8 -*-
```

## 标识符

标识符由字母、数字和下划线组成，但是第一个字符必须是字母或下划线。同时标识符对大小写敏感。
在 Python 3 中，可以用中文作为变量名，非 ASCII 标识符也是允许的了。比如如下常见的变量标识符

```python
abc = '12'
_var = 12
# 23var = 'test' #这个数字开头的不合法会提示错误
```

另外，对于一些关键字是python自己保留使用，也不能用它们来作为任何标识符。
Python 的标准库提供了一个 keyword 模块，可以输出当前版本的所有关键字：

```python
import keyword

print(keyword.kwlist)
```

输出结果为：

```
['False', 'None', 'True', 'and', 'as', 'assert', 'async', 'await', 'break', 'class',
'continue', 'def', 'del', 'elif', 'else', 'except', 'finally', 'for', 'from', 'global',
'if', 'import', 'in', 'is', 'lambda', 'nonlocal', 'not', 'or', 'pass', 'raise', 'return',
'try', 'while', 'with', 'yield']
```

## 注释

Python中单行注释以 `#` 开头，可以单独占一行，也可以写到语句后面

```python
# 第一个注释
print("Hello, Python!")  # 第二个注释
```

多行注释可以用多个 `#` 号，还可以用三引号 `'''` 和 `"""`：

```python
# 这是一个注释

'''
这是多行注释，用三个单引号
这是多行注释，用三个单引号
'''

"""
这是多行注释，用三个双引号
这是多行注释，用三个双引号 
"""
print("Hello, World!")
```

## 行与缩进

python最具特色的就是使用缩进来表示代码块，不需要使用大括号 {} 。

缩进的空格数是可变的，但是同一个代码块的语句必须包含相同的缩进空格数。但是强烈建议设置成一样空格数，
并且不要使用tab缩进，全部使用4个空格是一个良好的编程规范。

```python
if True:
    print("True")
else:
    print("False")
```

## 多行语句

Python 通常是一行写完一条语句，但如果语句很长，我们可以使用反斜杠(\)来实现多行语句，类似Shell脚本的语法

```python
total = item_one +
        item_two +
        item_three
```

如果是在 [], {}, 或 () 中的多行语句，不需要使用反斜杠(\)，例如：

```python
total = ['item_one', 'item_two', 'item_three',
         'item_four', 'item_five']
```

## 运算符

与 Java 一致，除了以下特例：

算法运算符

    ** 幂 - 返回x的y次幂
    / 除 - x 除以 y （返回小数） 在整数除法中，除法（/）总是返回一个浮点数
    // 取整除 - 返回商的整数部分

逻辑运算符：

    and 布尔"与" - 如果 x 为 False，x and y 返回 False，否则它返回 y 的计算值
    or 布尔"或" - 如果 x 是 True，它返回 x 的值，否则它返回 y 的计算值。
    not 布尔"非" - 如果 x 为 True，返回 False 。如果 x 为 False，它返回 True。

成员运算符：

    in 如果在指定的序列中找到值返回 True，否则返回 False。
    not in 如果在指定的序列中没有找到值返回 True，否则返回 False。

示例代码如下

```python
x = 9
y = 2
print(x ** y)  # 81
print(x / y)  # 4.5
print(x // y)  # 4

print(x and y)  # 2
print(x or y)  # 9
print(not x)  # False

z = [1, 2, 3]
print(x in z)  # False
print(x not in z)  # True
print(y in z)  # True
```

## 数字（Number）

Python 支持三种不同的数值类型：

    整型 int - 通常被称为是整型或整数，是正或负整数，不带小数点。Python3 整型是没有限制大小的。
    浮点型 float - 浮点型由整数部分与小数部分组成。
    复数 complex - 复数由实数部分和虚数部分构成，可以用 a + bj，或者 complex(a,b) 表示。

数字类型转换：

    int(x) 将 x 转换为一个整数。
    float(x) 将 x 转换到一个浮点数。
    complex(x) 将 x 转换到一个复数，实数部分为 x，虚数部分为 0。
    complex(x, y) 将 x 和 y 转换到一个复数，实数部分为 x，虚数部分为 y。

## 字符串（String）

字符串运算符：

    + 字符串连接
    * 重复输出字符串
    [] 通过索引获取字符串中字符
    [ : ] 截取字符串中的一部分
    in 如果字符串中包含给定的字符返回 True
    not in 如果字符串中不包含给定的字符返回 True
    r/R 原始字符串：所有的字符串都是直接按照字面的意思来使用，没有转义特殊或不能打印的字符
    % 格式字符串
    python 三引号允许一个字符串跨多行，字符串中可以包含换行符、制表符以及其他特殊字符。

python中单引号和双引号使用完全相同。
反斜杠可以用来转义，使用r可以让反斜杠不发生转义。 如`r"this is a line with \n"` 则\n会显示，并不是换行。

## 空行

函数之间或类的方法之间用空行分隔，空行并不是Python语法的一部分，作用在于分隔两段不同功能或含义的代码，
便于日后代码的维护或重构。要善于用空行来排版程序代码。

## 同一行多条语句

Python可以在同一行中使用多条语句，语句之间使用分号(;)分割

```python
import sys;

x = 'test';
sys.stdout.write(x + '\n')
```

## 代码块

缩进相同的一组语句构成一个代码块。像if、while、def和class这样的复合语句，首行以关键字开始，以冒号( : )结束，
该行之后的一行或多行代码构成代码组。我们将首行及后面的代码组称为一个子句(clause)。

```python
var = 'user1'
if var == 'user2':
    print('equal to user2')
    print('find user2')
else:
    print('not equal to user2')
    print('not find user2')
```

## import 与 from...import

Python使用 `import` 或者 `from...import` 来导入相应的模块。

* 将整个模块(somemodule)导入，格式为`import somemodule`
* 从某个模块中导入某个函数，格式为：`from somemodule import somefunction`
* 从某个模块中导入多个函数，格式为：`from somemodule import firstfunc, secondfunc`
* 将某个模块中的全部函数导入，格式为：`from somemodule import *`

导入 sys 模块

```python
import sys

print('============Python import mode=========');
print('命令行参数为:')
for i in sys.argv:
    print(i)
print('\n python 路径为', sys.path)
```

导入 sys 模块的 argv 和 path 成员

```python
from sys import argv, path  #  导入特定的成员

print('=========python from import===========')
print('path:', path)  # 因为已经导入path成员，所以此处引用时不需要加sys.path
```

## 模块

模块是一个包含所有你定义的函数和变量的文件，其后缀名是.py。
模块可以被别的程序引入，以使用该模块中的函数等功能。这也是使用 python 标准库的方法。

编写文件 myfunction.py：

```python
#!/usr/bin/python

def add(x):
    return x + 10
```

引用该模块：

```python
#!/usr/bin/python
import myfunction

print(myfunction.add(2))  # 结果输出12
```

## 函数

函数定义在模块中，函数代码块以 def 关键词开头，后接函数标识符名称和圆括号 ( )。
任何传入参数和自变量必须放在圆括号中间，圆括号之间可以用于定义参数。
函数的第一行语句可以选择性地使用文档字符串，用于存放函数说明。
函数内容以冒号起始，并且缩进。以 `return 表达式` 结束函数，选择性地返回一个值给调用方。
不带表达式的 return 相当于返回 None。比如如下定义了一个add函数，并且有一个参数

```python
#!/usr/bin/python

def add(x):
    return x + 10
```

## 条件和循环

这些跟C语言和Java一样，只不过Python语法是用`:`和缩进来控制，不适用大括号

下面是使用if语句进行条件判断：

```python
age = int(input("Input your age: "))

if age < 10:
    print('< 10')
elif age < 20:
    print('10 ~ 20')
else:
    print('> 20')
```

循环分成for和while两种类型循环。

for循环使用`for x in xxx`的语法，xxx表示一个可迭代对象，关于可迭代对象后续会讲解，
这里你只需要认为是一种可以产出一种序列的对象。

```python
sum = 0
for x in [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]:
    sum = sum + x
print(sum)
```

还有一种是通过`range()`函数生成序列，然后在序列上面做循环

```python
for x in range(5):
    print(x)
```

如果你还想在迭代的时候访问下标，则可以使用内置的`enumerate()`函数。它可以把一个list变成索引-元素对。

```python
for i, value in enumerate(['A', 'B', 'C']):
    print(i, value)
```

另外一种就是while循环，条件满足时候一直不断循环，条件不满足时退出循环。
比如我们要计算100以内所有奇数之和，可以用while循环实现：

```python
sum = 0
n = 99
while n > 0:
    sum = sum + n
    n = n - 2
print(sum)
```

## Python的main函数

一般如果临时测试python代码，可以写个main函数，直接执行该模块会执行main函数的方法，
如果是被别的模块导入的时候则不会执行这个main函数，可以很好地保护这个模块。一般写法如下：

```python
def foo():
    str = "function"
    print(str);


if __name__ == "__main__":
    print("main")
    foo()
```

好啦，Python大概的一些基本语法就介绍到这里。