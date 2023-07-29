---
title: 每天5分钟玩转Python（09） - 切片
date: '2019-06-09 09:56:22 +0800'
toc: true
categories: [ python ]
tags: [ python教程 ]
---

我们经常会遇见只取列表、元组或者字符串中一部分的场景。比如一个列表如下：

```python
_list = ['a', 'b', 'c', 'd', 'e']
```

如果只想取前面3个元素咋整。最笨的方法就是创建一个新的列表，通过下标引用来填充这个列表：

```python
sub_list = [_list[0], _list[1], _list[2]]
```

<!-- more -->

这样几个元素还好，如果几百个元素你还不得累死。
有人说我可以用循环啊，就像下面这样：

```python
sub_list = []
for i in range(3):
    sub_list.append(_list[i])
```

其实对于这种常见的操作，Python早就为我们准备好了倚天剑 - 切片操作。

通过使用切片，一行代码搞定上面的循环：

```python
sub_list = _list[0:3]
```

`_list[0:3]`表示从索引0开始，一直到3（不包含）为止。如果第一个索引为0，还可以省略，这样写效果也一样：

```python
sub_list = _list[:3]
```

如果最后一个索引是结尾，则后面的索引也可以省略，比如我想取全部的列表值，其实这个就是复制列表的效果。
则可以这样写：

```python
copy_list = _list[:]
```

另外，还支持从后往前取元素，最后一个元素的索引是-1，倒数第二个索引是-2，以此类推。
比如我要从倒数第二个开始，取3个元素。就这样写：

```python
sub_list = _list[-1:-4]
```

取最后3个元素：

```python
sub_list = _list[-3:]
```

除了顺序外，还可以加一个步长参数。默认步长是1。如果我想每2个取一次呢，可以这样写：

```python
sub_list = _list[0:5:2]
```

倒叙取的时候也能加上步长，比如逆序每隔2个取一个数，注意逆序的时候因为是每次往后退2步，那么步数应该写-2：

```python
sub_list = _list[-1::-2]
```

上面演示的都是对列表list的切片操作，其实对元组tuple、字符串str都可以适用。

```python
_tuple = ('a', 'b', 'c', 'd', 'e')
sub_tuple = _tuple[-1::-2]
_str = 'abcde'
sub_str = _str[-1::-2]
print(sub_list)
print(sub_str)
```

打印结果如下：

```
('e', 'c', 'a')
eca
```

对于切片的内部原理以及自定义切片等高级操作，我会在Python进阶教程中详细讲解。

