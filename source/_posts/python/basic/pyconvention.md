---
title: Python命名规约
date: '2018-08-05 10:02:33 +0800'
toc: true
categories: [ python ]
tags: [ python ]
---

应该避免的名称

* 单字符名称, 除了计数器和迭代器.
* 包/模块名中的连字符(-)
* 双下划线开头并结尾的名称(Python保留, 例如__init__)

<!-- more -->

## 命名约定

* 所谓"内部(Internal)"表示仅模块内可用, 或者, 在类内是保护或私有的.
* 用单下划线(_)开头表示模块变量或函数是protected的(使用from module import *时不会包含).
* 用双下划线(__)开头的实例变量或方法表示类内私有.
* 将相关的类和顶级函数放在同一个模块里. 不像Java, 没必要限制一个类一个模块.
* 对类名使用大写字母开头的单词(如CapWords, 即Pascal风格), 但是模块名应该用小写加下划线的方式(如lower_with_under.py).
  尽管已经有很多现存的模块使用类似于CapWords.py这样的命名, 但现在已经不鼓励这样做, 因为如果模块名碰巧和类名一致, 这会让人困扰.

## Python之父Guido推荐的规范

 Type                       | Public             | Internal                                                          
----------------------------|--------------------|-------------------------------------------------------------------
 Modules                    | lower_with_under   | _lower_with_under                                                 
 Packages                   | lower_with_under   | -                                                                 
 Classes                    | CapWords           | _CapWords                                                         
 Exceptions                 | CapWords           | -                                                                 
 Functions                  | lower_with_under() | _lower_with_under()                                               
 Global/Class Constants     | CAPS_WITH_UNDER    | _CAPS_WITH_UNDER                                                  
 Global/Class Variables     | lower_with_under   | _lower_with_under                                                 
 Instance Variables         | lower_with_under   | _lower_with_under (protected) or __lower_with_under (private)     
 Method Names               | lower_with_under() | _lower_with_under() (protected) or __lower_with_under() (private) 
 Function/Method Parameters | lower_with_under   | -                                                                 
 Local Variables            | lower_with_under   | -                                                                 

PyCharm里面规范检查，建议打开检查，严格按照规范编写和组织代码。
