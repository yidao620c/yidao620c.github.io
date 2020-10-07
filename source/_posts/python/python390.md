---
layout: post
title: Python3.9正式版发布
date: '2020-10-07 16:24:10 +0800'
toc: true
categories: python
tags:
  - python
---

Python3.9在2020年10月5号正式发布，官网release说明地址为<https://www.python.org/downloads/release/python-390/>

从这个版本开始，官方默认提供的Windows安装版本为64位的，同时不再支持Win7系统。所以童鞋们想用新版本的，就赶紧升级到Win10吧。

![](https://xnstatic-1253397658.file.myqcloud.com/python390.png)

<!--more-->

## 新特性说明

Python3.9带来了很多新特性，在release说明中列出了16个，这里只讲几个最酷的。

### 新语法特性

_PEP 584 - dict支持合并操作_

```python
dict1 = {'name': 'zhangsan', 'age': 22}
dict2 = {'gender': 'male'}
print(dict1 | dict2)
```

_PEP 585 - 泛型支持内置集合类_

在类型注解里面不再需要引入List和Dict类，可以直接使用内置的list和dict做泛型类型。
```python
def generic(param: list[str]) -> None:
    for var in param:
        print(var)
```

_PEP 614 - 对装饰器的限制放开_

任何规范的表达式都能当成装饰器来用。

### 新方法

_PEP 616 - str对象增加去除前缀和后缀的方法_

```python
def removeprefix(self: str, prefix: str, /) -> str:
    if self.startswith(prefix):
        return self[len(prefix):]
    else:
        return self[:]

def removesuffix(self: str, suffix: str, /) -> str:
    # suffix='' should not call self[:-0].
    if suffix and self.endswith(suffix):
        return self[:-len(suffix)]
    else:
        return self[:]
```

### 标准库新特性

_PEP 593 - 灵活的函数与变量注解_

通过Annotated实现函数和变量注解，具体说明参考[PEP593](https://www.python.org/dev/peps/pep-0593/)

_PEP 615 - 增加zoneinfo模块来支持`IANA Time Zone Database`_

### 新的解释器

_PEP 617 - CPython使用新的PEG解析器_

相比较原来的LL解析器性能差不多，但是新的PEG解析器更加灵活，可用来设计新的语法。

### 发布流程更新

_PEP 602 - 以年度为单位来作为发布周期_

主要版本大概每12个月发布一次，时间在每年的10月份左右。

小白鼠们可以行动了，下载来体验一把。
