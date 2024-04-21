---
title: python核心 - 异步IO
date: 2015-12-21 22:22:22 +0800
toc: true
categories: [ python ]
tags: [ python核心 ]
---

由于CPU的速度远远快于磁盘、网络等IO，我们可选择使用多进程或多线程来并发执行代码。
然而系统不能无限制增加线程，而且切换线程开销也大，一旦线程数量过多，CPU花的时间主要在切换线程上，导致性能下降。

另外一种解决方案是异步IO，当代码需要执行一个耗时的IO操作时，它只发出IO指令，并不等待IO结果，然后就去执行其他代码了。
一段时间后，当IO返回结果时，再通知CPU进行处理。

异步IO模型需要一个消息循环，在消息循环中，主线程不断地重复"读取消息-处理消息"这一过程：

```python
loop = get_event_loop()
while True:
    event = loop.get_event()
    process_event(event
```

息模型是如何解决同步IO必须等待IO操作这一问题的呢？当遇到IO操作时，代码只负责发出IO请求，不等待IO结果，
然后直接结束本轮消息处理，进入下一轮消息处理过程。当IO操作完成后，将收到一条"IO完成"的消息，
处理该消息时就可以直接获取IO操作结果。
<!-- more -->

## 协程

在学习异步IO模型前，我们先来了解协程。

协程，又称微线程，纤程。英文名Coroutine。一般的子程序，或者称为函数，在所有语言中都是层级调用，通过栈实现的，
一个线程就是执行一个子程序。子程序调用总是一个入口，一次返回，调用顺序是明确的。而协程的调用和子程序不同。
协程看上去也是子程序，但执行过程中，在子程序内部可中断，然后转而执行别的子程序，在适当的时候再返回来接着执行。

执行效率。因为子程序切换不是线程切换，而是由程序自身控制，因此，没有线程切换的开销，和多线程比，线程数量越多，
协程的性能优势就越明显。

Python对协程的支持是通过generator实现的。

在generator中，我们不但可以通过for循环来迭代，还可以不断调用next()函数获取由yield语句返回的下一个值。

但是Python的yield不但可以返回一个值，它还可以接收调用者发出的参数。

下面我们通过一个生产者-消费者模型来看看协程运行机制：

```python
#!/usr/bin/env python
# -*- coding: utf-8 -*-
"""
Topic: 协程
生产者-消费者实现
"""
import time


def consumer():
    r = ''
    while True:
        n = yield r
        if not n:
            return
        print('[CONSUMER] Consuming %s...' % n)
        time.sleep(1)
        r = '200 OK'


def producer(c):
    c.next()
    n = 0
    while n < 5:
        n += 1
        print('[PRODUCER] Producing %s...' % n)
        r = c.send(n)
        print('[PRODUCER] Consumer return: %s' % r)
    c.close()


if __name__ == '__main__':
    c = consumer()
    producer(c)
```

程序调用说明：

1. 首先调用c.next()启动生成器；
2. 然后，一旦生产了东西，通过c.send(n)切换到consumer执行；
3. consumer通过yield拿到消息，处理，又通过yield把结果传回；
4. producer拿到consumer处理的结果，继续生产下一条消息；
5. producer决定不生产了，通过c.close()关闭consumer，整个过程结束。

整个流程无锁，由一个线程执行，produce和consumer协作完成任务，所以称为"协程"，而非线程的抢占式多任务。

"子程序就是协程的一种特例。"说白了，协程其实就是一个单线程。子程序切换不是线程切换，而是由程序自身控制，
因此，没有线程切换的开销，和多线程比，线程数量越多，协程的性能优势就越明显。

协程是一个线程执行，那怎么利用多核CPU呢？最简单的方法是多进程+协程，既充分利用多核，
又充分发挥协程的高效率，可获得极高的性能。

python语言本身通过yield关键字只能是部分实现了协程，完全的协程实现可以参考优秀的第三方模块gevent

## asyncio

asyncio是Python 3.4版本引入的标准库，直接内置了对异步IO的支持。

asyncio的编程模型就是一个消息循环。我们从asyncio模块中直接获取一个EventLoop的引用，
然后把需要执行的协程扔到EventLoop中执行，就实现了异步IO。

用asyncio实现Hello world代码如下：

```python
import asyncio


@asyncio.coroutine
def hello():
    print("Hello world!")
    # 异步调用asyncio.sleep(1):
    r = yield from asyncio.sleep(1)
    print("Hello again!")


# 获取EventLoop:
loop = asyncio.get_event_loop()
# 执行coroutine
loop.run_until_complete(hello())
loop.close()
```

大致解释下执行流程：

`@asyncio.coroutine`把一个generator标记为coroutine类型，然后，我们就把这个coroutine扔到EventLoop中执行。

`hello()`会首先打印出`Hello world!`，然后，`yield from`语法可以让我们方便地调用另一个generator。
由于`asyncio.sleep()`也是一个coroutine，所以线程不会等待`asyncio.sleep()`，而是直接中断并执行下一个消息循环。
当`asyncio.sleep()`返回时，线程就可以从`yield from`拿到返回值（此处是None），然后接着执行下一行语句。

把`asyncio.sleep(1)`看成是一个耗时1秒的IO操作，在此期间，主线程并未等待，
而是去执行EventLoop中其他可以执行的coroutine了，因此可以实现并发执行。

我们用Task封装两个coroutine试试：

```python
import asyncio
import threading


@asyncio.coroutine
def hello():
    print('Hello world! (%s)' % threading.currentThread())
    yield from asyncio.sleep(1)
    print('Hello again! (%s)' % threading.currentThread())


loop = asyncio.get_event_loop()
tasks = [hello(), hello()]
loop.run_until_complete(asyncio.wait(tasks))
loop.close()
```

运行结果:

```
Hello world! (<_MainThread(MainThread, started 11168)>)
Hello world! (<_MainThread(MainThread, started 11168)>)
...中间暂停大约1秒...
Hello again! (<_MainThread(MainThread, started 11168)>)
Hello again! (<_MainThread(MainThread, started 11168)>)
```

由打印的当前线程名称可以看出，两个coroutine是由同一个线程并发执行的。

如果把asyncio.sleep()换成真正的IO操作，则多个coroutine就可以由一个线程并发执行。

我们用asyncio的异步网络连接来获取sina、sohu和163的网站首页：

```python
import asyncio


@asyncio.coroutine
def wget(host):
    print('wget %s...' % host)
    connect = asyncio.open_connection(host, 80)
    reader, writer = yield from connect
    header = 'GET / HTTP/1.0\r\nHost: %s\r\n\r\n' % host
    writer.write(header.encode('utf-8'))
    yield from writer.drain()
    while True:
        line = yield from reader.readline()
        if line == b'\r\n':
            break
        print('%s header > %s' % (host, line.decode('utf-8').rstrip()))
    # Ignore the body, close the socket
    writer.close()


loop = asyncio.get_event_loop()
tasks = [wget(host) for host in ['www.sina.com.cn', 'www.sohu.com', 'www.163.com']]
loop.run_until_complete(asyncio.wait(tasks))
loop.close()
```

运行结果：

```
wget www.sohu.com...
wget www.sina.com.cn...
wget www.163.com...
www.sina.com.cn header > HTTP/1.1 200 OK
www.sina.com.cn header > Content-Type: text/html
...省略...
www.163.com header > HTTP/1.0 200 OK
www.163.com header > Date: Sun, 15 Jan 2017 12:43:36 GMT
...省略...
www.sohu.com header > HTTP/1.1 200 OK
www.sohu.com header > Content-Type: text/html
...省略...
```

可见3个连接由一个线程通过coroutine并发完成。
asyncio提供了完善的异步IO支持，异步操作需要在coroutine中通过yield from完成，
多个coroutine可以封装成一组Task然后并发执行。

## async/await

为了简化并更好地标识异步IO，从Python 3.5开始引入了新的语法async和await，可以让coroutine的代码更简洁易读。

async和await是针对coroutine的新语法，上面的例子重新改写成：

```python
async def hello():
    print("Hello world!")
    r = await asyncio.sleep(1)
    print("Hello again!")
```

剩下的代码保持不变。

## aiohttp

asyncio可以实现单线程并发IO操作。如果仅用在客户端，发挥的威力不大。如果把asyncio用在服务器端，例如Web服务器，
由于HTTP连接就是IO操作，因此可以用单线程+coroutine实现多用户的高并发支持。

asyncio实现了TCP、UDP、SSL等协议，aiohttp则是基于asyncio实现的HTTP框架。

我们先安装aiohttp：

```
pip install aiohttp
```

然后编写一个HTTP服务器，分别处理以下URL：

```
/ - 首页返回b'<h1>Index</h1>'；
/hello/{name} - 根据URL参数返回文本hello, %s!。
```

代码如下：

```python
import asyncio

from aiohttp import web


async def index(request):
    await asyncio.sleep(0.5)
    return web.Response(body=b'<h1>Index</h1>')


async def hello(request):
    await asyncio.sleep(0.5)
    text = '<h1>hello, %s!</h1>' % request.match_info['name']
    return web.Response(body=text.encode('utf-8'))


async def init(loop):
    app = web.Application(loop=loop)
    app.router.add_route('GET', '/', index)
    app.router.add_route('GET', '/hello/{name}', hello)
    srv = await loop.create_server(app.make_handler(), '127.0.0.1', 8000)
    print('Server started at http://127.0.0.1:8000...')
    return srv


loop = asyncio.get_event_loop()
loop.run_until_complete(init(loop))
loop.run_forever()
```

注意aiohttp的初始化函数init()也是一个coroutine，loop.create_server()则利用asyncio创建TCP服务。
