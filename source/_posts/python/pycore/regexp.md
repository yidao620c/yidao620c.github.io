---
title: python核心 - 正则表达式
date: 2015-12-12 22:22:22 +0800
toc: true
categories: [ python ]
tags: [ python核心 ]
---

字符串是编程时涉及到的最多的一种数据结构，对字符串进行操作的需求几乎无处不在。

正则表达式是一种用来匹配字符串的强有力的武器。它的设计思想是用一种描述性的语言来给字符串定义一个规则，
凡是符合规则的字符串，我们就认为它"匹配"了，否则，该字符串就是不合法的。

我们判断一个字符串是否是一个合法的电话号码分两步，首先创建一个符合电话号码规则的正则表达式，
然后用这个正则表达式来匹配这个字符串是否合法。
<!-- more -->

## 基础

因为正则表达式也是用字符串表示，我们就要先学习如何用字符来描述字符。

正则表达式中直接给出字符就是精确匹配，而用\d可匹配数字，\w匹配一个字母或数字。

* '00\d'可以匹配'006'，但无法匹配'00A'；
* '\d\d\d'可以匹配'123'；
* '\w\w\d'可以匹配'py3'；

.可以匹配任意一个字符，所以

* 'py.'可以匹配'pyc'、'pyo'、'py!'等等

在正则表达式中匹配变长的字符：

* *表示任意个字符（包括0个）
* +表示至少一个字符
* ?表示0个或1个字符
* {n}表示n个字符
* {n,m}表示n-m个字符

来看一个复杂的例子：\d{3}\s+\d{3,8}

我们来从左到右解读一下：

* \d{3}表示匹配3个数字，例如'010'；
* \s可以匹配一个空格（也包括Tab等空白符），所以\s+表示至少有一个空格，例如匹配' '，' '等；
* \d{3,8}表示3-8个数字，例如'1234567'。

综合起来，上面的正则表达式可以匹配以任意个空格隔开的带区号的电话号码。

如果要匹配'.010-12345'这样的号码呢？由于'.'是特殊字符，
在正则表达式中，要用'\'转义，所以，上面的正则是'\.\d{3}-\d{3,8}'

## 更多正则式

做更精确地匹配，可以用[]表示范围，还有些事定位字符串位置：

* [0-9a-zA-Z_]可以匹配一个数字、字母或者下划线；
* [a-zA-Z_][0-9a-zA-Z_]{0, 19}更精确地限制了变量的长度是1-20个字符（前面1个字符+后面最多19个字符）。
* A|B可以匹配A或B，所以(P|p)ython可以匹配'Python'或者'python'
* ^表示行的开头，^\d表示必须以数字开头
* $表示行的结束，\d$表示必须以数字结束。

所以，py也可以匹配'python'但不匹配' python'，但是加上^py$就变成了整行匹配，
就只能匹配'py'了，注意从字符串开始匹配。

下面列举一些正则表达式里的元字符及其作用

 元字符   | 说明                         
-------|----------------------------
 .     | 代表任意字符                     
 \     | 转义                         
 []    | 匹配内部的任一字符或子表达式             
 [^]   | 对字符集和取非                    
 -     | 定义一个区间                     
 *     | 匹配前面的字符或者子表达式0次或多次         
 *?    | 惰性匹配上一个                    
 +     | 匹配前一个字符或子表达式一次或多次          
 +?    | 惰性匹配上一个                    
 ?     | 匹配前一个字符或子表达式0次或1次重复        
 {n}   | 匹配前一个字符或子表达式               
 {m,n} | 匹配前一个字符或子表达式至少m次至多n次       
 {n,}  | 匹配前一个字符或者子表达式至少n次          
 {n,}? | 前一个的惰性匹配                   
 ^     | 匹配字符串的开头，MULTILINE下匹配每一行开头 
 $     | 匹配字符串结束，MULTILINE下匹配下一行前面  
 \A    | 匹配字符串开头                    
 \Z    | 匹配字符串结尾                    
 \c    | 匹配一个控制字符                   
 \b    | 匹配单词边界                     
 \t    | 匹配制表符                      
 \s    | 匹配任意空白符[\t\r\n\f\v]        
 \S    | 匹配任意非空白符[^ \t\r\n\f\v]     
 \d    | 匹配任意数字                     
 \D    | 匹配数字以外的字符                  
 \w    | 匹配任意数字字母下划线                
 \W    | 不匹配数字字母下划线                 
 \1    | 匹配内容和group 1一样             

## re模块

python通过re模块提供对正则表达式的完全支持。由于Python的字符串本身也用\转义，
所以在构造正则表达式字符串的时候，强烈建议使用原始字符串，也就是带r前缀：

```python
s = r'ABC\.001'  # Python的字符串
# 对应的正则表达式字符串不变：
# 'ABC\.001'
```

### 匹配

```python
import re

if re.match(r'^\d{3}-\d{3,8}$', '010-12345'):
    print('OK')
else:
    print("failed")
```

match()方法判断是否匹配，如果匹配成功，返回一个Match对象，否则返回None。

### 切分字符串

用正则表达式切分字符串比用固定的字符更灵活，请看正常的切分代码：

```python
'a b   c'.split(' ')
['a', 'b', '', '', 'c']
```

无法识别连续的空格，用正则表达式试试：

```python
re.split(r'\s+', 'a b   c')
['a', 'b', 'c']
```

无论多少个空格都可以正常分割。加入,试试：

```python
re.split(r'[\s\,]+', 'a,b, c  d')
['a', 'b', 'c', 'd']
```

### 分组

除了简单地判断是否匹配之外，正则表达式还有提取子串的强大功能。用()表示的就是要提取的分组（Group）。比如

^(\d{3})-(\d{3,8})$分别定义了两个组，可以直接从匹配的字符串中提取出区号和本地号码：

```python
>> > m = re.match(r'^(\d{3})-(\d{3,8})$', '010-12345')
>> > m
< _sre.SRE_Match
object;
span = (0, 9), match = '010-12345' >
>> > m.group(0)
'010-12345'
>> > m.group(1)
'010'
>> > m.group(2)
'12345'
```

如果正则表达式中定义了组，就可以在Match对象上用group()方法提取出子串来。

注意到group(0)永远是原始字符串，group(1)、group(2)……表示第1、2、……个子串。

### 贪婪匹配

最后需要特别指出的是，正则匹配默认是贪婪匹配，也就是匹配尽可能多的字符。举例如下，匹配出数字后面的0

```python
re.match(r'^(\d+)(0*)$', '102300').groups()
('102300', '')
```

由于\d+采用贪婪匹配，直接把后面的0全部匹配了，结果0*只能匹配空字符串了。

必须让\d+采用非贪婪匹配（也就是尽可能少匹配），才能把后面的0匹配出来，加个?就可以让\d+采用非贪婪匹配。

```python
re.match(r'^(\d+?)(0*)$', '102300').groups()
('1023', '00')
```

当我们需要在字符串中查找符合正则式的字符串时，可以使用search来代替match，因为match永远是从开头开始匹配，
而search可以从中间开始匹配。

```python
# 贪婪匹配
data = "Sat Mar 21 09:20:57 2009::spiepu@ovwdmrnuw.com::1237598457-6-9"
# 获取最后的那三个连字符连起来的三个数，
# 搜索比匹配更合适，因为不在开头
patt = r'\d+-\d+-\d+'
print(re.search(patt, data).group())  # 打印出  1237598457-6-9
# 使用匹配，必须用到group
patt = r'.+(\d+-\d+-\d+)'
print(re.match(patt, data).group(1))  # 打印出  7-6-9，知道贪婪的厉害了吧
# 接下来使用非贪婪操作符?
patt = r'.+?(\d+-\d+-\d+)'
print(re.match(patt, data).group(1))  # 打印出  1237598457-6-9
# 只获取三个数的中间那个数字：
patt = r'-(\d+)-'
print(re.search(patt, data).group())  # 打印-6-
print(re.search(patt, data).group(1))  # 打印6
```

### 预编译

当我们在Python中使用正则表达式时，re模块内部会干两件事情：

1. 编译正则表达式，如果正则表达式的字符串本身不合法，会报错；
2. 用编译后的正则表达式去匹配字符串。

如果一个正则表达式要重复使用几千次，出于效率的考虑，我们可以预编译该正则表达式，
接下来重复使用时就不需要编译这个步骤了，直接匹配

```python
# 日期的正则式
datepat = re.compile(r'(\d+)/(\d+)/(\d+)')
datepat.match('2016/12/12').groups()
```

编译后生成RE对象，由于该对象自己包含了正则表达式，所以调用对应的方法时不用给出正则字符串

### 正则替换

前面讲过匹配用match，搜索用search，现在讲替换用sub。

基本用法：

```python
# 第一种预编译
pat = re.compile('a')
b = pat.sub('A', 'abcasd')
# 第二种直接使用
b = re.sub('a', 'A', 'abcasd')
```

两种用法和之前的match一样，就是在字符串'abcasd'中利用正则式'a'搜索，将搜索到的字符串全部替换成'A'

关于替换还有更多高级主题，还可以自定义替换规则，比如只替换第3个出现的。下面我用示例代码展示：

```python
#!/usr/bin/env python
# -*- encoding: utf-8 -*-
"""
Topic: 正则式分组替换示例
"""
import re


class Nth(object):
    """
    如果 sub 函数的第二个参数是个函数，则每次匹配到的时候都会执行这个函数。
    函数接受匹配到的那个 match object 作为参数，返回用来替换的字符串。
    利用这个特性就可以只在第 N 次匹配的时候返回要替换成的字符串，其他时候原样返回不做替换即可。
    """

    def __init__(self, nth, replacement):
        self.nth = nth
        self.replacement = replacement
        self.calls = 0

    def __call__(self, matchobj):
        self.calls += 1
        if self.calls == self.nth:
            return self.replacement
        return matchobj.group(0)


def re_sub():
    a = re.sub(r'(foo)(bar)', r'\g<1>123\g<2>', 'foobar')
    print(a)

    a = re.sub('a', 'A', 'abcasd')  # 找到a用A替换，后面见和group的配合使用
    pat = re.compile('a')
    b = pat.sub('A', 'abcasd')
    print(b)

    # 通过组进行更新替换：
    pat = re.compile(r'(www\.)(.*)(\..{3})')  # 正则表达式
    print(pat.match('www.dxy.com').group(2))
    # 通过正则匹配找到符合规则的"www.dxy.com" ，取得组2字符串，用baidu替换之
    print('-----------')
    print(pat.sub(r'\g<1>baidu\g<3>', 'hello,www.dxy.com'))

    pat = re.compile(r'(\w+) (\w+)')
    s = 'hello world ! hello hz !'
    pat.findall('hello world ! hello hz !')
    # [('hello', 'world'), ('hello', 'hz')]
    # 通过正则得到组1(hello)，组2(world)，再通过sub去替换。即组1替换组2，组2替换组1，调换位置。
    print(pat.sub(r'\2 \1', s))

    # 替换字符串中第3个出现的good
    pat = re.compile(r'(good)')
    a = pat.sub(Nth(3, 'bad'), 'This is a good story, good is good. Oh, good')
    print(a)
    # 传入一个lambda函数，在匹配处两边加双引号
    a = pat.sub(lambda m: '"' + m.group(0) + '"', 'This is a good story, very good.')
    print(a)

    # 前后匹配，特殊构造，不作为分组
    # 前向定界（只能写固定宽度的正则式）：(?<=...)之前的字符串需要匹配表达式才能成功匹配
    # 后向定界(可以写任意正则式)：(?=...)之后的字符串需要匹配表达式才能成功匹配
    pat = re.compile(r'(?<=(a){1} )good(?= (story){1,2})')
    print(pat.sub('bad', 'This is a good story, very good.'))
    # 所以如果想定位前面字符为可变长字符串时，需要使用到组
    pat = re.compile(r'(a{1,2} )(good)(?= (story){1,2})')
    print(pat.sub(lambda m: m.group(1) + 'bad', 'This is a good story, very good.'))


if __name__ == '__main__':
    pp = re.compile(r'((http|https|ftp)://[a-zA-Z0-9+\-&@#/%?=~_|!:,.;]*[a-zA-Z0-9+\-&@#/%=~_|])')
    aa = 'one： http://www.baidu.com/ two'
    print(pp.sub('<a href="\g<1>">', aa))
    # print(pp.findall(aa))
    # for m in pp.finditer(aa):
    #     print(m.group())

```

## 高级用法

先来详细讲解扩展用法(?...)，一般来讲这个扩展不会创建新的分组group， (?P<name>...)是唯一例外。

### 不可捕获组

(?:...)，匹配括号中的正则式，但是不能再后面通过组来捕获这个东西了，
无论是在正则式里面还是在match对象或替换中都不能通过组来捕获。

### 命名组

用法为(?P<name>...)，这样定义后可以在后面通过名字来应用这个组。
比如匹配单引号或双引号引起来的字符串：(?P<quote>['"]).*?(?P=quote)

三种场景使用

 场景         | 用法                                
------------|-----------------------------------
 正则式本身      | (?P=quote) 或 \1                   
 match对象m   | m.group('quote') 或 m.end('quote') 
 re.sub()替换 | \g<quote> 或 \g<1> 或 \1            

### lookahead匹配

等于用法为Hello (?=World)，匹配"Hello "，当且仅当紧跟着World的时候，匹配完不消耗字符串World。

不等于用法为Hello (?!World)，匹配"Hello "，当且仅当后面不是紧跟着World的时候，匹配完不消耗字符串World。

### lookbehind匹配

等于用法为(?<=Hello )World，匹配"World"，当且仅当前面紧跟着"Hello "，匹配不消耗Hello 。

不等于用法为(?<!Hello )World，匹配"World"，当且仅当前面不紧跟着"Hello "，匹配不消化Hello 。

### 条件匹配

(?(id/name)yes-pattern|no-pattern)

如果group id或name存在就匹配yes-pattern否则匹配no-pattern，no-pattern可以省略。
比如,`(<)?(\w+@\w+(?:\.\w+)+)(?(1)>|$)`
匹配email: `aa@gmail.com`或`<aa@gmail.com>`，但不匹配`<aa@gmail.com`

注：上面使用不可捕获组语法(?:\.\w+)+，所以这个正则式只有两个group。永远记住上面写过的(?...)不创建分组group

### flags标志

re模块提供很多flags标志来影响匹配行为

用法：

```python
patt = re.compile('[a-zA-Z]+', flags=re.M)
m = patt.match('xxx')

# 多个flags
m = re.match('[a-zA-Z]+', 'xxx', flags=re.I | re.M)

re.search(pattern, string, flags=0)
re.fullmatch(pattern, string, flags=0)
re.split(pattern, string, maxsplit=0, flags=0)
re.findall(pattern, string, flags=0)
re.sub(pattern, repl, string, count=0, flags=0)

```

    re.I/IGNORECASE:    忽略大小写
    re.M/MULTILINE:     多行模式，主要影响的是^和$
    re.S/DOTALL:        .可表示换行符

关于正则表达式要讲的内容太多了，这里我介绍了python中最常用的功能，其他更加深入的主题，请参考专业书籍。

