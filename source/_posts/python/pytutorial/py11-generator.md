---
title: 每天5分钟玩转Python（11） - 生成器（上）
date: '2019-06-11 21:27:10 +0800'
toc: true
categories: [ python ]
tags: [ python教程 ]
---

这篇开始要展示python这门语言真正的魅力所在了。python有一些高级功能，
让我们的代码写起来超级爽，所以才会有这么多人喜欢它。

这篇先介绍生成器这个东东，学会你就知道它有多强大了，不过对于生成器的讲解稍微有点长，
可能看完不止5分钟，所以我分了上下两篇。建议读者耐心看完，这是进阶的必经之路。
<!-- more -->

## 列表推导

在讲生成器之前，先讲讲python里面常用的几个常见的推导式：

*列表推导式（list comprehension）*

```python
my_list = [f(x) for x in sequence if cond(x)]
```

返回一个新的列表，列表中每个元素是原来列表中满足`cond(x)`条件的元素经过函数`f(x)`转换后的值。

*字典推导式（dictionary comprehension）*

```python
my_dict = {k(x): v(x) for x in sequence if cond(x)}
```

*集合推导式（set comprehension）*

```python
my_set = {f(x) for x in sequence if cond(x)}
```

几种推导式原理都一样，我只讲列表推导就可以了。它是python内置用来生成新列表的语法。
除了上面的单层循环，你还能使用双层循环。比如打印9*9乘法表。一句话搞定：

```python
print("\n".join("\t".join(["{} * {} = {}".format(y, x, x * y) for y in range(1, x + 1)]) for x in range(1, 10)))
```

1. `"\t".join(["{} * {} = {}".format(y, x, x*y) for y in range(1, x+1)])`
   这句是一个将以推导式的形式生成的列表转换成字符串，以制表符分隔；单独这一句代码不能执行，
   因为x还未赋值，我们可以假设它有一个值，这一行代码最终结果是打印99乘法表的每一行。
2. 剩下的代码则是外层循环，列表推导式，转换字符串，以换行符分隔，跟第１步一样。
   注意外层的join操作对象并没有再使用`[]`转成列表，加上也可以。

输出结果：

```
1 * 1 = 1
1 * 2 = 2	2 * 2 = 4
1 * 3 = 3	2 * 3 = 6	3 * 3 = 9
1 * 4 = 4	2 * 4 = 8	3 * 4 = 12	4 * 4 = 16
1 * 5 = 5	2 * 5 = 10	3 * 5 = 15	4 * 5 = 20	5 * 5 = 25
1 * 6 = 6	2 * 6 = 12	3 * 6 = 18	4 * 6 = 24	5 * 6 = 30	6 * 6 = 36
1 * 7 = 7	2 * 7 = 14	3 * 7 = 21	4 * 7 = 28	5 * 7 = 35	6 * 7 = 42	7 * 7 = 49
1 * 8 = 8	2 * 8 = 16	3 * 8 = 24	4 * 8 = 32	5 * 8 = 40	6 * 8 = 48	7 * 8 = 56	8 * 8 = 64
1 * 9 = 9	2 * 9 = 18	3 * 9 = 27	4 * 9 = 36	5 * 9 = 45	6 * 9 = 54	7 * 9 = 63	8 * 9 = 72	9 * 9 = 81
```

## 生成器的优势

列表推导会将所有元素都生成后填充到列表中，这样当元素非常多的时候就会很耗内存。
如果列表元素不需要一次性就初始化，而是在使用它的时候再算出来，这就是经常说的懒加载技术。
这种一遍循环一遍计算元素值得机制称为生成器（generator）。

## 创建生成器

创建生成器的方式有两种方式。

第一种方式是将列表推导的`[]`改成`()`就可以了。比如

```python
_list = [x for x in range(6)]
print(_list)

_list_generator = (x for x in range(6))
print(_list_generator)
```

打印结果如下：

```
[0, 1, 2, 3, 4, 5]
<generator object <genexpr> at 0x0000029AC6221820>
```

注意下面的是一个对象，类型为`generator`

另外一种方式是通过`yield`关键字定义。如果在python的函数中出现了一个`yield`关键字，
那么python不会把它当做普通函数看待了，而是将它当做是一个生成器generator。
比如我定义一个斐波拉契数列的函数：

```python
def fib(max):
    n, a, b = 0, 0, 1
    while n < max:
        yield b
        a, b = b, a + b
        n = n + 1
    yield 'done'
```

## 使用生成器

拿到生成器对象后，我怎么使用这个生成器呢？很简单，可以通过不断的调用next()方法来获取它的下一个元素：

```python
_list_generator = (x for x in range(3))
print(next(_list_generator))
print(next(_list_generator))
print(next(_list_generator))
print(next(_list_generator))
```

结果如下：

```
0
1
2
Traceback (most recent call last):
  File "/core-python/ch02_function/my_slice.py", line 43, in <module>
    print(next(_list_generator))
StopIteration
```

怎么到了第四行出现一个异常了呢？因为我把它所有的元素都消耗完了，只有3个元素。第四次再访问就会抛`StopIteration`异常出来。

这一篇讲了生成器的基本用法，下一篇再继续深入讲解`yield`关键字以及python中的协程原理。
