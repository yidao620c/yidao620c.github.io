---
title: 每天5分钟玩转Python（06） - 基本数据类型（中）
date: '2019-06-06 17:22:35 +0800'
toc: true
categories: [ python ]
tags: [ python教程 ]
---

继续上一篇的数据结构讲解，这篇讲解列表List（列表）和Tuple（元组）的使用方法。
<!-- more -->

List（列表）

列表应该是Python中使用最频繁的数据类型了。列表可以完成大多数集合类的数据结构实现。
列表中元素的类型可以不相同，它支持数字，字符串甚至可以包含列表（所谓嵌套）。
列表是写在方括号 `[]` 之间、用逗号分隔开的元素列表。

```python
list1 = ['abcd', 123, 3.14, 'test', True, 70.2 + 3.2j]
list2 = [555, '666']
```

### 索引访问

列表是有序集合，因此要访问列表的任何元素，只需将该元素的位置或索引告诉Python即可。
要访问列表元素，可指出列表的名称，再指出元素的索引，注意索引从0开始，并将其放在方括号内。

```python
print(list1[2])  # 将打印3.14
```

列表是可变数据类型，所以还能添加、修改和删除元素。

### 修改元素

修改列表元素的语法与访问列表元素的语法类似。要修改列表元素，可指定列表名和要修改
的元素的索引，再指定该元素的新值。
例如，假设有一个摩托车列表，其中的第一个元素为'honda'，如何修改它的值呢？

```python
motorcycles = ['honda', 'yamaha', 'suzuki']
print(motorcycles)

motorcycles[0] = 'ducati'
print(motorcycles)
```

### 添加元素

Python提供了多种在既有列表中添加新数据的方式

**1、在列表末尾添加元素**

方法append()可以将元素添加到列表末尾，而不会影响列表中其他元素

```python
motorcycles = ['honda', 'yamaha', 'suzuki']
print(motorcycles)
motorcycles.append('ducati')
print(motorcycles)
```

**2、在列表中插入元素**

使用方法insert()可在列表的任何位置添加新元素。为此，你需要指定新元素的索引和值。

```python
motorcycles = ['honda', 'yamaha', 'suzuki']
motorcycles.insert(1, 'ducati')
```

会在索引为1的地方插入元素'ducati'，最后的列表如下

```
['honda', 'ducati', 'yamaha', 'suzuki']
```

### 删除元素

删除列表元素有两种方式，你可以根据位置或值来删除列表中的元素。

如果知道要删除的元素在列表中的位置，可使用del语句。

```python
motorcycles = ['honda', 'yamaha', 'suzuki']
print(motorcycles)

del motorcycles[0]  # 删除列表第一个元素
print(motorcycles)
```

如果你想删除列表末尾的元素，并且还想使用被删除的元素，可以使用`pop()`方法。
学过数据结构的同学应该都知道，这个时候列表就像一个栈，而删除列表末尾的元素相当于弹出栈顶元素。

```python
motorcycles = ['honda', 'yamaha', 'suzuki']
print(motorcycles)

popped_motorcycle = motorcycles.pop()
print(motorcycles)
print(popped_motorcycle)
```

实际上，你可以使用pop()来删除列表中任何位置的元素，只需在括号中指定要删除的元素的索引即可。

```python
motorcycles = ['honda', 'yamaha', 'suzuki']
second_owned = motorcycles.pop(1)
```

有时候，你不知道要从列表中删除的值所处的位置。如果你只知道要删除的元素的值，可使用方法remove()。

```python
motorcycles = ['honda', 'yamaha', 'suzuki', 'ducati']
print(motorcycles)

motorcycles.remove('ducati')
print(motorcycles)
```

### 列表截取

这个跟字符串类似，你可以只截取列表的一部分来访问。列表截取的语法格式如下：

```
变量[开始下标:结束下标]
```

注意，截取的范围包含开始下标，但是不包含结束下标。索引值以 0 为开始值，-1 为从末尾的开始位置。
如果不填写开始下标则表示从最开始截取，如果不填写结束下标则表示截取到最后。

![](https://xnstatic-1253397658.file.myqcloud.com/p06_truncate.png)

```python
list = ['abcd', 123, 3.14, 'test', True, 70.2 + 3.2j]
list2 = [555, '666']

print(list)  # 输出完整列表
print(list[0])  # 输出列表第一个元素
print(list[1:3])  # 从第二个开始输出到第三个元素
print(list[2:])  # 输出从第三个元素开始的所有元素
print(list2 * 2)  # 输出两次列表
print(list + list2)  # 连接列表
```

Python 列表截取可以接收第三个参数，参数作用是截取的步长，
以下实例在索引1到索引4的位置并设置为步长为 2（间隔一个位置）来截取字符串：

![](https://xnstatic-1253397658.file.myqcloud.com/p06_step.png)

如果第三个参数为负数表示逆向读取，以下实例用于翻转字符串：

```python
def reverseWords(input):
     
    # 通过空格将字符串分隔符，把各个单词分隔为列表
    inputWords = input.split(" ")
 
    # 翻转字符串
    # 假设列表 list = [1,2,3,4],  
    # list[0]=1, list[1]=2 ，而-1表示最后一个元素。list[-1]=4(与list[3]=4一样)
    # inputWords[-1::-1] 有三个参数
    # 第一个参数 -1 表示最后一个元素
    # 第二个参数为空，表示移动到列表末尾
    # 第三个参数为步长，-1 表示逆向
    inputWords=inputWords[-1::-1]
    # 重新组合字符串
    output = ' '.join(inputWords)
    return output
 
if __name__ == "__main__":
    input = 'I like runoob'
    rw = reverseWords(input)
    print(rw)
```

## Tuple（元组）

元组（tuple）与列表类似，不同之处在于元组的元素不能修改。元组写在小括号 () 里，元素之间用逗号隔开。
元组中的元素类型也可以不相同。

```python
tuple1 = ('abcd', 123, 3.14, 'test', True, 70.2 + 3.2j)
tuple2 = (555, '666')

print(tuple1)  # 输出完整元组
print(tuple1[0])  # 输出元组的第一个元素
print(tuple1[1:3])  # 输出从第二个元素开始到第三个元素
print(tuple1[2:])  # 输出从第三个元素开始的所有元素
print(tuple2 * 2)  # 输出两次元组
print(tuple1 + tuple2)  # 连接元组
```

元组与列表类似，可以被索引且下标索引从0开始，-1 为从末尾开始的位置。也可以进行截取，
跟上面讲的规则一样这里就不重复说了。

构造包含 0 个或 1 个元素的元组比较特殊，所以有一些额外的语法规则：

```python
tup1 = ()  # 空元组
tup2 = (20,)  # 一个元素，需要在元素后添加逗号
```

由于元组是不可变对象，那么就只有访问元素和截断操作了。不能对元组进行添加、更新、删除的操作。

好啦，列表和元组基本用法介绍完了。喝杯可乐歇会吧。
