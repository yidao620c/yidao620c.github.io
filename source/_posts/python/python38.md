---
layout: post
title: 一分钟了解Python3.8新特性
date: '2019-10-15 19:02:42 +0800'
toc: true
categories: python
tags:
  - python
abbrlink: 24109
---

今天Python3.8版本刚刚发布，添加了很多新功能。这里介绍几个我觉得最酷的特性，感受一下小Python的美。
更全面的特性请直接去官网看：<https://docs.python.org/3.8/whatsnew/3.8.html>

## 赋值表达式

新增一个新的赋值语法`:=`，它将赋给左边变量并将赋值语句转换成一个表达式。该特性编号为
[PEP 572](https://www.python.org/dev/peps/pep-0572)就是赋值表达式，也叫“海象运算符”。
可以看到这个操作符看上去像一只横躺着的海象。

比如下面这个例子中，使用赋值表达式可以防止你调用两次`len()`方法
``` python
if (n := len(a)) > 10:
    print(f"List is too long ({n} elements, expected <= 10)")
```

## 强制位置参数

在函数的参数中增加一个`/`可以强制前面的参数必须使用位置参数调用，不能使用关键字参数。
比如下面的函数中，a跟b必须使用位置参数调用，c和d随便，e和f必须使用关键字参数
``` python
def f(a, b, /, c, d, *, e, f):
    print(a, b, c, d, e, f)
```

更多请参考[PEP 570](https://www.python.org/dev/peps/pep-0570)

## 运行时的审计钩子

[PEP 578](https://www.python.org/dev/peps/pep-0578) 添加了一个审计钩子和一个确认开放钩子。

``` python
import sys
import urlib.request

def audit_hook(event, args):
    if event in ['urllib.Request']:
        print(f'Network {event=} {args=}')

sys.addaudithook(audit_hook
```

目前支持审计的事件名字和API可以看PEP文档， urllib.Request是其中之一。另外还可以自定义事件。

## 跨进程内存共享

在multiprocessing添加了一个shared_memory模块，支持跨进程间的内存共享。

## 全新第三方包读取模块

新模块`importlib.metadata`提供了读取第三方包原信息的能力。比如它能从第三方包中读取版本号、实体列表
``` python
>>> # Note following example requires that the popular "requests"
>>> # package has been installed.
>>>
>>> from importlib.metadata import version, requires, files
>>> version('requests')
'2.22.0'
>>> list(requires('requests'))
['chardet (<3.1.0,>=3.0.2)']
>>> list(files('requests'))[:5]
[PackagePath('requests-2.22.0.dist-info/INSTALLER'),
 PackagePath('requests-2.22.0.dist-info/LICENSE'),
 PackagePath('requests-2.22.0.dist-info/METADATA'),
 PackagePath('requests-2.22.0.dist-info/RECORD'),
 PackagePath('requests-2.22.0.dist-info/WHEEL')]
```

## functools改进

现在functools.lru_cache()注解可以不用加括号了，下面的两种写法都支持
``` python
@lru_cache
def f(x):
    pass

@lru_cache(maxsize=256)
def f(x):
    pass
```

## unittest改进

增加了一个AsyncMock支持异步mock，并增加了一些新的断言函数。
新增了`addModuleCleanup()` 和 `addClassCleanup()`，对应的用来清理`setUpModule()`和`setUpClass()`

同时还支持对于协程的模拟测试，只需要测试用例继承自`unittest.IsolatedAsyncioTestCase`

``` python
import unittest


class TestRequest(unittest.IsolatedAsyncioTestCase):

    async def asyncSetUp(self):
        self.connection = await AsyncConnection()

    async def test_get(self):
        response = await self.connection.get("https://example.com")
        self.assertEqual(response.status_code, 200)

    async def asyncTearDown(self):
        await self.connection.close()


if __name__ == "__main__":
    unittest.main()
```

## venv

venv现在包含了一个`Activate.ps1`的脚本，可以在PowerShell Core 6.1环境中激活虚拟环境。

## math

math模块中中添加了一个函数`math.dist()`用来计算两点之间的欧几里德距离。

`math.hypot()`现在可以处理多维数据，之前仅支持2-D场景。

新增一个`math.prod()`函数，跟累积求和类似，是一个累积乘积函数。
``` python
>>> prior = 0.8
>>> likelihoods = [0.625, 0.84, 0.30]
>>> math.prod(likelihoods, start=prior)
0.126
```