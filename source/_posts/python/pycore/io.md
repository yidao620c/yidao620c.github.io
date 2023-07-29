---
title: python核心 - IO编程
date: 2015-12-20 22:22:22 +0800
toc: true
categories: [ python ]
tags: [ python核心 ]
---

IO在计算机中指Input/Output，也就是输入和输出。由于程序和运行时数据是在内存中驻留，
由CPU这个超快的计算核心来执行，涉及到数据交换的地方，通常是磁盘、网络等，就需要IO接口。

通常，程序完成IO操作会有Input和Output两个数据流。当然也有只用一个的情况，
比如，从磁盘读取文件到内存，就只有Input操作，反过来，把数据写到磁盘文件里，就只是一个Output操作。

Stream（流）是一个很重要的概念，可以把流想象成一个水管，数据就是水管里的水，但是只能单向流动。
Input Stream就是数据从外面（磁盘、网络）流进内存，Output Stream就是数据从内存流到外面去。
<!-- more -->

由于CPU和内存的速度远远高于外设的速度，所以IO操作分两种方式：

1. 第一种是CPU等着，也就是程序暂停执行后续代码，等待IO操作完成后再接着往下执行，这种模式称为同步IO；
2. 另一种方法是CPU不等待，只是告诉磁盘"您老慢慢操作，我接着干别的事去了"，于是，后续代码可以立刻接着执行，这种模式称为异步IO。

很明显，使用异步IO来编写程序性能会远远高于同步IO，但是异步IO的缺点是编程模型复杂。
因为异步IO需要知道什么时候完成了，有很多种方式，比如回调模式，轮询模式等。

本篇的IO编程都是同步模式，异步IO单独一篇讲解。

## 文件读写

读写文件是最常见的IO操作，分为读取文本文件和读取二进制文件。我们演示基本使用方法：

```python
def basic_rw():
    with open('/path/to/file', 'r', encoding='utf-8', errors='ignore') as f:
        print(f.read(200))  # 一次读取200字节bytes
        print(f.read())  # 一次性读完
    with open('/Users/michael/test.txt', 'w', encoding='utf-8') as f:
        f.write('Hello, world!')
    # 读取二进制文件
    with open('/Users/michael/test.jpg', 'rb') as f:
        f.read()
    # 写二进制文件
    with open('/Users/michael/test.jpg', 'rw') as f:
        f.write(b"hello world.")
```

## StringIO和BytesIO

python动态语言有个很有用的特性就是鸭子类型，无需类型强制要求，只需要看起来像鸭子就认为是鸭子了。

对应到这里，就是在python中凡是定义了read()/write()方法的对象就成为file-like Object，
可像正常文件操作一样操作它们。比如典型的StringIO和BytesIO，都是在内存中读写，并不涉及真正的磁盘操作。

```python
def stringio_rw():
    f = StringIO()
    f.write('hello')
    f.write(' ')
    f.write('中文!')
    print(f.getvalue())

    f = StringIO('Hello!\nHi!\n中文!')
    while True:
        s = f.readline()
        if not s:
            break
        print(s.strip())


def bytesio_rw():
    f = BytesIO()
    f.write('中文'.encode('utf-8'))
    print(f.getvalue())
```

## 文件和目录

如果我们要操作文件、目录，可以在命令行下面输入操作系统提供的各种命令来完成。比如ls、cp等命令。

Python内置的os模块也可以直接调用操作系统提供的接口函数。
操作文件和目录的函数一部分放在os模块中，一部分放在os.path模块中，这一点要注意一下

```python
def file_dir():
    # 查看当前目录的绝对路径
    print(os.path.abspath('.'))
    # 在某个目录下创建一个新目录，首先把新目录的完整路径表示出来
    os.path.join('/Users/michael', 'testdir')
    # 然后创建一个目录
    os.mkdir('/Users/michael/testdir')
    # 删掉一个目录:
    os.rmdir('/Users/michael/testdir')
    # 分开路径和文件名
    os.path.split('/Users/michael/testdir/file.txt')
    # 得到文件扩展名
    os.path.splitext('/path/to/file.txt')
    # 重命名
    os.rename('test.txt', 'test.py')
    # 删除文件
    os.remove('test.py')
    # 列出当前目录所有文件夹
    all_dir = [x for x in os.listdir('.') if os.path.isdir(x)]
    # 列出所有py文件
    all_py = [x for x in os.listdir('.') if os.path.isfile(x) and os.path.splitext(x)[1] == '.py']
```

还有些功能比如复制操作cp在os模块中并未定义，不过我们可以通过shutil模块来达到目的。
shutil模块中找到很多实用函数，它们可以看做是os模块的补充

## 序列化

我们把变量从内存中变成可存储或传输的过程称之为序列化，在Python中叫pickling，
在其他语言中也被称之为serialization，marshalling，flattening等等，都是一个意思。

反过来，把变量内容从序列化的对象重新读到内存里称之为反序列化，即unpickling。

Python提供了pickle模块来实现序列化。

```python
def _pickle():
    d = dict(name='Bob', age=20, score=88)
    result = pickle.dumps(d)  # 结果是一个字节
    dd = picle.loads(result)  # 通过字节反序列化
    print(dd)
    # 序列化到文件必须是二进制写入，不能文本形式
    with open('d:/dump.txt', 'wb') as f:
        pickle.dump(d, f)
    # 反序列化必须是二进制读取
    with open('d:/dump.txt', 'rb') as f:
        d = pickle.load(f)
        print(d)
```

如果我们要在不同的编程语言之间传递对象，就必须把对象序列化为标准格式，最好的方法是序列化为JSON

JSON表示的对象就是标准的JavaScript语言的对象，JSON和Python内置的数据类型对应如下：

 JSON类型	    | Python类型   
------------|------------
 {}         | dict       
 []         | list       
 "string"   | str        
 1234.56    | int或float  
 true/false | True/False 
 null       | None       

Python内置的json模块提供了非常完善的Python对象到JSON格式的转换。

```python
def dict2student(d):
    return Student(d['name'], d['age'], d['score'])


class Student(object):
    def __init__(self, name, age, score):
        self.name = name
        self.age = age
        self.score = score


def _json():
    """
    import json
    d = dict(name='Bob', age=20, score=88)
    json.dumps(d)  # 返回json字符串
    with open('d:/dump.txt', 'w', encoding='utf-8') as f:
        json.dump(d, f)
    json_str = '{"age": 20, "score": 88, "name": "Bob"}'
    dd = json.loads(json_str)
    print(dd)
    with open('d:/dump.txt', 'r', encoding='utf-8') as f:
        ddd = json.load(f)
        print(ddd)

    # 对象json序列化
    s = Student('Bob', 20, 88)
    # 因为通常class的实例都有一个__dict__属性，它就是一个dict，用来存储实例变量
    j_str = json.dumps(s, default=lambda obj: obj.__dict__)
    print(j_str)
    # 对象json反序列化
    print(json.loads(j_str, object_hook=dict2student))
```

更高级和复杂的对象json序列化使用第三方模块jsonpickle： <http://jsonpickle.github.io/>

## fileinput模块

经常需要对文本按行处理，或做正则替换，搜索之类的，这时候用fileinput模块会很方便。

【基本格式】:

```python
fileinput.input([files[, inplace[, backup[, bufsize[, mode[, openhook]]]]]])
```

【默认参数】:

```python
fileinput.input(files=None, inplace=False, backup='', bufsize=0, mode='r', openhook=None)
```

【参数说明】:

```
files:                  #文件的路径列表，默认是stdin方式，多文件['1.txt','2.txt',...]
inplace:                #是否将标准输出的结果写回文件，默认不取代
backup:                 #备份文件的扩展名，只指定扩展名，如.bak。如果该文件的备份文件已存在，则会自动覆盖。
bufsize:                #缓冲区大小，默认为0，如果文件很大，可以修改此参数，一般默认即可
mode:                   #读写模式，默认为只读
openhook:               #该钩子用于控制打开的所有文件，比如说编码方式等;
```

【常用函数】:

```python
fileinput.input()  # 返回能够用于for循环遍历的对象
fileinput.filename()  # 返回当前文件的名称
fileinput.lineno()  # 返回当前已经读取的行的数量（或者序号）
fileinput.filelineno()  # 返回当前读取的行的行号
fileinput.isfirstline()  # 检查当前行是否是文件的第一行
fileinput.isstdin()  # 判断最后一行是否从stdin中读取
fileinput.close()  # 关闭队列
```

下面通过几个简单的例子来说明使用方法

实例1：利用fileinput读取一个文件所有行

```python
for line in fileinput.input('data.txt', openhook=fileinput.hook_encoded("utf-8")):
    print(line.rstrip())
```

实例2：读取文件名，行，内容

```python
for line in fileinput.input('data.txt'):
    print(fileinput.filename(), '|', 'Line Number:',
          fileinput.lineno(), '|: ', line)
```

实例3：利用fileinput对多文件操作，并原地修改内容

```python
def process(line):
    return line.rstrip() + ' line'


for line in fileinput.input(['1.txt', '2.txt'], inplace=1):
    print(process(line))
```

实例4：利用fileinput实现文件内容替换，并将原文件作备份

```python
for line in fileinput.input('data.txt', inplace=1):
    if line[-2:] == "\r\n":
        print(line.rstrip() + "\n")
    else:
        print(line.rstrip())
```

实例5：利用fileinput批处理文件

```python
for line in fileinput.input(glob.glob("d*.txt"), inplace=1):
    if fileinput.isfirstline():
        print('-' * 20, 'Reading %s...' % fileinput.filename(), '-' * 20)
    print(str(fileinput.lineno()) + ': ' + line.rstrip().upper())
```

更多实例可以去我的github主页上面看..
