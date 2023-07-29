---
title: python核心 - 字符串编码
date: 2015-10-24 10:06:22 +0800
toc: true
categories: [ python ]
tags: [ python核心 ]
---

字符串也是一种数据类型，但是，字符串比较特殊的是还有一个编码问题。

Unicode把所有语言都统一到一套编码里，这样就不会再有乱码问题了。Unicode标准也在不断发展，
但最常用的是用两个字节表示一个字符（如果要用到非常偏僻的字符，就需要4个字节）。现代操作系统和大多数编程语言都直接支持Unicode。

现在，捋一捋ASCII编码和Unicode编码的区别：ASCII编码是1个字节，而Unicode编码通常是2个字节。

但是，如果你写的文本基本上全部是英文的话，用Unicode编码比ASCII编码需要多一倍的存储空间，在存储和传输上就十分不划算。
<!-- more -->

所以，本着节约的精神，又出现了把Unicode编码转化为"可变长编码"的UTF-8编码。
UTF-8编码把一个Unicode字符根据不同的数字大小编码成1-6个字节，
常用的英文字母被编码成1个字节，汉字通常是3个字节，只有很生僻的字符才会被编码成4-6个字节。
如果你要传输的文本包含大量英文字符，用UTF-8编码就能节省空间：

在计算机内存中，统一使用Unicode编码，当需要保存到硬盘或者需要传输的时候，就转换为UTF-8编码。

用记事本编辑的时候，从文件读取的UTF-8字符被转换为Unicode字符到内存里，
编辑完成后，保存的时候再把Unicode转换为UTF-8保存到文件：

![](https://xnstatic-1253397658.file.myqcloud.com/pystr001.png)

浏览网页的时候，服务器会把动态生成的Unicode内容转换为UTF-8再传输到浏览器：

![](https://xnstatic-1253397658.file.myqcloud.com/pystr002.png)

所以你看到很多网页的源码上会有类似<meta charset="UTF-8" />的信息，表示该网页正是用的UTF-8编码

### Python的字符串

搞清楚了令人头疼的字符编码问题后，我们再来研究Python的字符串。

Python 2.x版本虽然支持Unicode，但在语法上需要'xxx'和u'xxx'两种字符串表示方式。
在Python 3.x版本中，把'xxx'和u'xxx'统一成Unicode编码，即写不写前缀u都是一样的，
而以字节形式表示的字符串则必须加上b前缀：b'xxx'

在最新的Python 3版本中，字符串是以Unicode编码的，也就是说，Python的字符串支持多语言，例如：

```python
>> > print('包含中文的str')
包含中文的str
```

由于Python的字符串类型是str，在内存中以Unicode表示，一个字符对应若干个字节。如果要在网络上传输，
或者保存到磁盘上，就需要把str变为以字节为单位的bytes。

Python对bytes类型的数据用带b前缀的单引号或双引号表示：

```python
x = b'ABC'
```

要注意区分'ABC'和b'ABC'，前者是str，后者虽然内容显示得和前者一样，但bytes的每个字符都只占用一个字节。

以Unicode表示的str通过encode()方法可以编码为指定的bytes，例如：

```python
>> > 'ABC'.encode('ascii')
b'ABC'
>> > '中文'.encode('utf-8')
b'\xe4\xb8\xad\xe6\x96\x87'
>> > '中文'.encode('ascii')
Traceback(most
recent
call
last):
File
"<stdin>", line
1, in < module >
UnicodeEncodeError: 'ascii'
codec
can
't encode characters in position 0-1: ordinal not in range(128)
```

纯英文的str可以用ASCII编码为bytes，内容是一样的，含有中文的str可以用UTF-8编码为bytes。
含有中文的str无法用ASCII编码，因为中文编码的范围超过了ASCII编码的范围，Python会报错。

反过来，如果我们从网络或磁盘上读取了字节流，那么读到的数据就是bytes。要把bytes变为str，就需要用decode()方法：

```python
>> > b'ABC'.decode('ascii')
'ABC'
>> > b'\xe4\xb8\xad\xe6\x96\x87'.decode('utf-8')
'中文'
```

在操作字符串时，我们经常遇到str和bytes的互相转换。为了避免乱码问题，应当始终坚持使用UTF-8编码对str和bytes进行转换。

由于Python源代码也是一个文本文件，所以，当你的源代码中包含中文的时候，在保存源代码时，
就需要务必指定保存为UTF-8编码。当Python解释器读取源代码时，为了让它按UTF-8编码读取，我们通常在文件开头写上这两行：

```python
#!/usr/bin/env python
# -*- coding: utf-8 -*-
```

第一行注释是为了告诉Linux/OS X系统，这是一个Python可执行程序，Windows系统会忽略这个注释；

第二行注释是为了告诉Python解释器，按照UTF-8编码读取源代码，否则，你在源代码中写的中文输出可能会有乱码。

申明了UTF-8编码并不意味着你的.py文件就是UTF-8编码的，必须并且要确保文本编辑器正在使用UTF-8 without BOM编码：
![](https://xnstatic-1253397658.file.myqcloud.com/pystr003.png)

如果.py文件本身使用UTF-8编码，并且也申明了# -*- coding: utf-8 -*-，打开命令提示符测试就可以正常显示中文

### Python2和3的差异

先准备两张图片

![](https://xnstatic-1253397658.file.myqcloud.com/unicode005.png)

第一张图片是几个特殊字符的unicode代码点（CODE POINT），
用\uXXXX或者\xXX来表示，其中X都是十六进制字符，而\x表示前面一个字节为00就变成简写了。

![](https://xnstatic-1253397658.file.myqcloud.com/unicode006.png)

第二张图片是UTF-8怎样来通过可变字节来编码相应字符的代码点

在python2中

* str: a sequence of bytes
* unicode: a sequence of code points

在 Python2 中，有两种字符串数据类型。一种纯旧式的文字: "str" 对象,存储 bytes 。
如果你使用一个 "u" 前缀，那么你会有一个 "unicode" 对象，存储的是 code points 。
在一个 unicode 字符串中，你可以使用\u或\x来插入任何的 unicode 代码点(\x后面接2位的十六进制表示\u00XX这样的简写)。

```python
my_string = "Hello World."
print(type(my_string))

my_unicdoe = u"Hi \u2119\u01b4\u2602\u210c\xf8\u1f24"
print(len(my_unicode))  # 9个字符

my_utf8 = my_unicode.encode('utf-8')
print(len(my_utf8))  # 19个字节

my_unicode2 = my_utf8.decode('utf-8')
print(len(my_unicode2))  # 9个字符，恢复原来的

# my_unicode2.encode('ascii')  # 报错，因为ascii只能表示0-127个字符

my_unicode2.encode('ascii', 'replace')  # 用?代替不能编码的字符
my_unicode2.encode('ascii', 'xmlcharrefreplace')  # 对于不能编码的产生一个完全替代的 HTML/XML 字符，保护所有的原始数据
my_unicode2.encode('ascii', 'ignore')  # 忽略不能编码的字符，留下的都是ascii字符

```

在python3中

* str: a sequence of code points
* bytes: a sequence of bytes

1. 只有一个文本类型是`str`(默认就是unicode编码字符)
2. 有两个字节类型是`bytes`和`bytearray`

在 Python 2 中的 "str" 现在叫做 "bytes"，而 Python 2 中的 "unicode" 现在叫做 "str"。

```python
my_string = "Hi \u2119\u01b4\u2602\u210c\xf8\u1f24"
print(type(my_string))  # <class 'str'>

my_bytes = b"Hello world"
print(type(my_bytes))  # <class 'bytes'>
```

避免bytes和unicode混合使用报错的原则是：制造一个 Unicode 三明治， bytes 在外， Unicode 在内。

我们有五个不可忽视的事实:

* 程序中所有的输入和输出均为 byte
* 世界上的文本需要比 256 更多的符号来表现
* 你的程序必须处理 byte 和 unicode
* byte 流中不会包含编码信息
* 指明的编码有可能是错误的

### 常见乱码问题

1. SyntaxError: Non-ASCII character
2. UnicodeDecodeError: 'ascii' codec can't decode byte 0xe5 in position 108
3. UnicodeEncodeError: 'gb2312' codec can't encode character u'\u2014' in position 72366:

字符串在Python内部的表示是unicode 编码，因此，在做编码转换时，通常需要以unicode作为中间编码，
即先将其他编码的字符串解码（decode）成unicode，再从unicode编码（encode）成另一种编码。

decode的作用是将其他编码的字符串转换成unicode编码，如str1.decode('gb2312')，表示将gb2312编码的字符串str1转换成unicode编码。
encode的作用是将unicode编码转换成其他编码的字符串，如str2.encode('gb2312')，表示将unicode编码的字符串str2转换成gb2312编码。

```python
#!/usr/bin/env python
# coding=utf-8
s = "中文"
if isinstance(s, unicode):
    # s=u"中文"
    print
    s.encode('gb2312')
else:
    # s="中文"
    print
    s.decode('utf-8').encode('gb2312')
```

这是你在编程中保持 Unicode 清洁的三个建议:

1. Unicode 三明治：尽可能的让你程序处理的文本都为 Unicode 。
2. 了解你的字符串。你应该知道你的程序中，哪些是 unicode, 哪些是 byte, 对于这些 byte 串，你应该知道，他们的编码是什么。
3. 测试 Unicode 支持。使用一些奇怪的符号来测试你是否已经做到了以上几点。

如果你遵循以上建议的话，你将会写出对 Unicode 支持很好的代码。不管 Unicode 中有多么不规整的编码你的程序也不会挂掉。

### 参考

1. <http://lucumr.pocoo.org/2014/1/5/unicode-in-2-and-3/>
2. <http://pycoders-weekly-chinese.readthedocs.io/en/latest/issue5/unipain.html>

