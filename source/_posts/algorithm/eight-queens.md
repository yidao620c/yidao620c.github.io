---
title: 回溯法解决八皇后问题
date: 2015-04-15 17:11:42 +0800
toc: true
categories: [ 算法之美 ]
tags: [ 算法 ]
---

八皇后问题是一个以国际象棋为背景的问题：如何能够在8×8的国际象棋棋盘上放置八个皇后，
使得任何一个皇后都无法直接吃掉其他的皇后？为了达到此目的，任两个皇后都不能处于同一条横行、纵行或斜线上。
八皇后问题可以推广为更一般的n皇后摆放问题：这时棋盘的大小变为n×n，而皇后个数也变成n。当且仅当n = 1或n ≥ 4时问题有解
— 摘自[八皇后问题wiki](http://zh.wikipedia.org/wiki/%E5%85%AB%E7%9A%87%E5%90%8E%E9%97%AE%E9%A2%98)
<!-- more -->

利用回溯法解决

```python
# coding: utf-8
__author__ = 'Administrator'


# 冲突函数
# 如果下一个皇后和正在考虑的前一个皇后的水平距离为0，
# 或者等于垂直距离（在一条对角线上），返回True
def conflict(state, nextX):
    nextY = len(state)
    for i in range(nextY):
        if abs(state[i] - nextX) in (0, nextY - i):
            return True
    return False


# num：皇后的总数
# state：已经注册了的每行的皇后的位置列表（X坐标）
def queens(num=8, state=()):
    print(state)
    find = False
    for pos in range(num):
        if not conflict(state, pos):
            find = True
            if len(state) == num - 1:
                # 如果state大小已经是num-1了，那么yield最后一个位置
                yield (pos,)
            else:
                for result in queens(num, state + (pos,)):
                    yield (pos,) + result
    if not find:
        print("HO, NO..." + str(state))


print(list(queens(4)))
```

### 回溯法原理

打个比方，想象一下你要出席一个很重要的会议，但你不知道在哪儿开会，在你面前有两扇门，
开会地点就在其中一扇门后面，于是有人挑了左边的进入，然后又发现两扇门，后来再选了左边的门，
结果却错了，于是回溯到刚才的两扇门那里，并选择右边的门，结果还是错的，
于是再次回溯，直到回到开始点，在那里选择了右边的门。

比较难理解的是这两行代码：

```python
for result in queens(num, state + (pos,)):
```

递归的调用queens生成器，将state + (pos)当做参数传给queens，state是已经存在的合法状态列表，
而pos是对要操作的该行进行遍历的位置参数，请注意state + (pos) 并没有将它赋值给state，
是因为如果后面的返回没有合法值的话可以回溯到前面的状态，也就是说这次迭代里面我并没有说pos是合法状态，
只是可能是合法状态，那么当往下的迭代无合法值，一级一级的跳出的时候，每次跳到一级，这个state都是合法的状态。

```python
yield (pos,) + result：
```

这行的意思是我将当前状态pos作为前缀再加上一直到到底的迭代位置序列，也就是正序排列位置。
就好比是这样的调用：(0+(1+(2+(3+(…..)))))，这样够清楚了吧。^_^

