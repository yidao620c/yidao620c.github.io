---
title: 每天5分钟玩转Python（05） - 基本数据类型（上）
date: '2019-06-05 20:10:35 +0800'
toc: true
categories: [ python ]
tags: [ python教程 ]
---

在介绍数据类型之前，有必要先讲一下变量赋值语法。

Python 中的变量不需要声明。每个变量在使用前都必须赋值，变量赋值以后该变量才会被创建。
变量并没有类型，我们所说的"类型"是变量所指的内存中对象的类型。你可以认为变量就是指向内存中对象的一个指针。

使用等号（=）用来给变量赋值。等号（=）运算符左边是一个变量名,等号（=）运算符右边是存储在变量中的值。

```python
counter = 100  # 整型变量
miles = 1000.0  # 浮点型变量
name = "test"  # 字符串
```

<!-- more -->

Python允许你同时为多个变量赋值。例如：

```python
a = b = c = 1
```

也可以为多个对象指定多个变量。例如：

```python
a, b, c = 1, 2, "test"
```

## 标准数据类型

Python3 中有六个标准的数据类型：

1. Number（数字）
1. String（字符串）
1. List（列表）
1. Tuple（元组）
1. Set（集合）
1. Dictionary（字典）

Python3 的六个标准数据类型中

    不可变数据（3 个）：Number（数字）、String（字符串）、Tuple（元组）；
    可变数据（3 个）：List（列表）、Dictionary（字典）、Set（集合）。

接下来分别介绍这6种数据类型

### Number（数字）

Python3 的数字类型支持 int、float、bool、complex（复数）。只有一种整数类型 int，表示为长整型。
内置的 type() 函数可以用来查询变量所指的对象类型。

```python
a, b, c, d = 20, 5.5, True, 4 + 3j
print(type(a), type(b), type(c), type(d))
```

输出结果如下

```
<class 'int'> <class 'float'> <class 'bool'> <class 'complex'>
```

此外还可以用 isinstance 来判断：

```python
isinstance(a, int)
```

数字运算例子：

```python
print(5 + 4)  # 加法
print(4.3 - 2)  # 减法
print(3 * 7)  # 乘法
print(2 / 4)  # 除法，得到一个浮点数
print(2 // 4)  # 除法，得到一个整数
print(17 % 3)  # 取余 
print(2 ** 5)  # 乘方
```

Python中的二进制、八进制、十六进制数表示

* 二进制使用`0b`开头比如0b10101100
* 八进制使用`0o`开头，注意是数字0+字母o。比如0o123
* 十六进制使用`ox`开头，比如0x23AB

科学计数法：这个跟其他语言基本一致，使用e表示10的指数运算。比如15e+18，表示15乘以10的18次方

数字类型例子

 int    | float      | complex    
--------|------------|------------
 10     | 0.0        | 3.14j      
 100    | 15.20      | 45.j       
 -788   | -21.9      | 9.322e-36j 
 080    | 32.3e+18   | .876j      
 -0490  | -90.       | -.6545+0J  
 -0x260 | -32.54e100 | 3e+26J     
 0x69   | 70.2E-12   | 4.53e-7j   

### String（字符串）

Python中的字符串用单引号 `'` 或双引号 `"` 括起来。

```python
a = 'test'
b = "test"
```

同时使用反斜杠 `\` 转义特殊字符，如果你不想让让反斜杠发生转义，可以在字符串前面加`r`，表示原始字符串。
下面的b和c效果一样。

```python
a = 'test\ntest'
b = 'test\\ntest'
c = r'test\ntest'
```

同时字符串跟后面讲的列表一样，可以进行截断操作。就是只读取一部分。索引值以 0 为开始值，-1 为从末尾的开始位置。

```python
str = 'XiongNeng'

print(str)  # 输出字符串
print(str[0:-1])  # 输出第一个到倒数第二个的所有字符
print(str[0])  # 输出字符串第一个字符
print(str[2:5])  # 输出从第三个开始到第五个的字符
print(str[2:])  # 输出从第三个开始的后的所有字符
print(str * 2)  # 输出字符串两次
print(str + "TEST")  # 连接字符串
```

暂时先介绍了3个基本数据类型，每天进步一小步就可以啦。
