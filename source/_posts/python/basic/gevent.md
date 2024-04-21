---
title: 浅谈coroutine和gevent
date: 2016-01-02 08:16:18 +0800
toc: true
categories: [ python ]
tags: [ python, gevent ]
---

一般将coroutine称之为协程（或微线程，也有称纤程的）。
我在python并发编程那篇文章已经详细讲解了进程Process和线程Thread的用法，
很早就想再写一篇专门讲解coroutine以及相关的优秀库gevent。

目前常见的coroutine应用都是网络程序中，所以我们先来看看各种不同的网络框架模型，
然后再介绍coroutine就会比较理解了。
<!-- more -->

## 不同网络模型

### 阻塞式单进程

这样的网络程序最简单，一个进程一个循环处理网络请求，当然性能也是最差的。
现在服务器采用这种模型的基本看不到了。

### 阻塞式多进程

每个请求开一个进程去处理，这样就能同时处理多个请求了，
不过缺点就是当请求数变大时CPU花在进程切换开销巨大，效率低下。

### 非阻塞式事件驱动

也是多进程，不过使用一个主进程循环来检查是否有网络I/O事件发生，再来决定怎样处理。
省去了上下文切换、进程复制等成本，也不会有死锁、竞争的发生。
不过缺点是没有阻塞式进程直观，[Twisted](https://twistedmatrix.com/trac/)就是这样的网络框架。

### 非阻塞式Coroutine

有没有可能既有事件驱动的好处，又有阻塞式编程的直观的模型呢？答案也行就是我要讲的coroutine了。
基本上它的本质也是事件驱动，只在单一循环上面检查事件的发生，但是加上了coroutine的概念。
[gevent](http://www.gevent.org/)就是这种框架的代表。

## Coroutine

简单来讲，coroutine就是允许你暂时中断之后再继续执行的程序。事实上，python最基础的coroutine就是生成器。

```python
#!/usr/bin/env python
# -*- coding: utf8 -*-

def foo():
    for i in range(10):
        # 暂时返回一个值，并将控制权交出去
        yield i
        print(u'foo: 控制权又回到我手上了，叫我大笨蛋')


bar = foo()
# 执行coroutine
print(next(bar))
print(u'main: 现在控制权在我手上，啦啦啦')
print('main:hello baby!')
# 回到刚刚foo这个coroutine中断的地方继续执行
print(next(bar))
print(u'main: 老大我又来了')
print(next(bar))
```

执行结果：

```
0
main: 现在控制权在我手上，啦啦啦
main:hello baby!
foo: 控制权又回到我手上了，叫我大笨蛋
1
main: 老大我又来了
foo: 控制权又回到我手上了，叫我大笨蛋
2
```

看到没有，我们的foo在执行过程中被中断又继续了好几回。
你刚开始可能觉得这样做没什么好处呀，thread的上下文切换不是也可以暂停再继续么。
其实重点在于：

* thread之间需要context-switch，而且成本很高，但coroutine没有任何上下文切换
* 可轻松生成大量的coroutine，基本没什么开销
* 所有的coroutine全部都在一个线程中完成
* thread的切换基本靠OS来觉得该执行哪个，而coroutine是自己控制。

## Gevent

优秀的第三方库gevent就是基于coroutine的Python网络库，
它用到Greenlet提供的，封装了libevent事件循环的高层同步API，它的coroutine是由Greenlet负责生成的。

事实上程序写的跟普通的阻塞式程序一样，但它千真万确是异步的，这是它神奇的地方，我们来看实际的例子:

```python
"""Spawn multiple workers and wait for them to complete"""

urls = ['http://www.gevent.org/', 'http://www.baidu.com', 'http://www.python.org']

import gevent
from gevent import monkey

# patches stdlib (including socket and ssl modules) to cooperate with other greenlets
monkey.patch_all()

import urllib2


def print_head(url):
    print（'Starting %s' % url）
    data = urllib2.urlopen(url).read()
    print（'%s: %s bytes: %r' % (url, len(data), data[:50])）

    jobs = [gevent.spawn(print_head, url) for url in urls]

    gevent.joinall(jobs)
```

写起来非常简单，不过里面有一句`monkey.patch_all()`会让人有点疑惑。
也就是猴子补丁，因为python内置的各种函数库和IO库一般都是阻塞式的，比如`sleep()`就会当前进程，
而monkey就是负责将这些阻塞函数全部取代替换成gevent中相应的异步函数。

gevent打了monkey patch之后会设置python相应的模块设置成非阻塞，
然后在内部实现epoll的机制，一旦调用非阻塞的IO(比如recv)都会立刻返回，
并且设置一个回调函数，这个回调函数用于切换到当前子coroutine，
设置好回掉函数之后就把控制权返回给主coroutine，主coroutine继续调度。
一旦网络I/O准备就绪，epoll会触发之前设置的回调函数，从而引发主coroutine切换到子coroutine，做相应的操作。

`joinall()`的意思是等待列表中所有coroutine完成后再返回。

### 上下文切换

gevent里面的上下文切换是非常平滑的。在下面的例子程序中，我们可以看到两个上下文通过调用 gevent.sleep()来互相切换。

```python
import gevent


def foo():
    print('Running in foo')
    gevent.sleep(0)
    print('Explicit context switch to foo again')


def bar():
    print('Explicit context to bar')
    gevent.sleep(0)
    print('Implicit context switch back to bar')


gevent.joinall([
    gevent.spawn(foo),
    gevent.spawn(bar),
])
```

接下来一个例子中可以看到gevent是安排各个任务的执行的：

```python
import gevent
import random


def task(pid):
    """
    Some non-deterministic task
    """
    gevent.sleep(random.randint(0, 2) * 0.001)
    print('Task', pid, 'done')


def synchronous():
    for i in range(1, 10):
        task(i)


def asynchronous():
    threads = [gevent.spawn(task, i) for i in xrange(10)]
    gevent.joinall(threads)


print('Synchronous:')
synchronous()

print('Asynchronous:')
asynchronous()
```

在同步的情况下，任务是按顺序执行的，在执行各个任务的时候会阻塞主线程。

而 `gevent.spawn` 的重要功能就是封装了greenlet里面的函数。
初始化的greenlet放在了threads这个list里面，
被传递给了 `gevent.joinall` 这个函数，它会阻塞当前的程序来执行所有的greenlet。

在异步执行的情况下，所有任务的执行顺序是完全随机的。每一个greenlet的都不会阻塞其他greenlet的执行。

### 生成greenlet

上面我们通过`gevent.spawn`来包装了对于greenlet的生成，
另外我们还能可以通过创建Greenlet的子类，并且重写 _run 方法来实现：

```python
import gevent
from gevent import Greenlet


class MyGreenlet(Greenlet):

    def __init__(self, message, n):
        Greenlet.__init__(self)
        self.message = message
        self.n = n

    def _run(self):
        print(self.message)
        gevent.sleep(self.n)


g = MyGreenlet("Hi there!", 3)
g.start()
g.join()
```

### Greenlet 的状态

greenlet在执行的时候也会出错。Greenlet有可能会无法抛出异常，停止失败，或者消耗了太多的系统资源。

greenlet的内部状态通常是一个依赖时间的参数。greenlet有一些标记来让你能够监控greenlet的状态

* started -- 标志greenlet是否已经启动
* ready -- 标志greenlet是否已经被终止
* successful() -- 标志greenlet是否已经被终止，并且没有抛出异常
* value -- 由greenlet返回的值
* exception -- 在greenlet里面没有被捕获的异常

```python
import gevent


def win():
    return 'You win!'


def fail():
    raise Exception('You fail at failing.')


winner = gevent.spawn(win)
loser = gevent.spawn(fail)

print(winner.started)  # True
print(loser.started)  # True

# Exceptions raised in the Greenlet, stay inside the Greenlet.
try:
    gevent.joinall([winner, loser])
except Exception as e:
    print('This will never be reached')

print(winner.value)  # 'You win!'
print(loser.value)  # None

print(winner.ready())  # True
print(loser.ready())  # True

print(winner.successful())  # True
print(loser.successful())  # False

# The exception raised in fail, will not propogate outside the
# greenlet. A stack trace will be printed to stdout but it
# will not unwind the stack of the parent.

print(loser.exception)
```

### 终止程序

在主程序收到一个SIGQUIT 之后会阻塞程序的执行让Greenlet无法继续执行。
这会导致僵尸进程的产生，需要在操作系统中将这些僵尸进程清除掉。

```python
import gevent
import signal


def run_forever():
    gevent.sleep(1000)


if __name__ == '__main__':
    gevent.signal(signal.SIGQUIT, gevent.shutdown)
    thread = gevent.spawn(run_forever)
    thread.join()
```

### 超时

在gevent中支持对于coroutine的超时控制，还能使用`with`上下文，示例：

```python
import gevent
from gevent import Timeout

time_to_wait = 5  # seconds


class TooLong(Exception):
    pass


with Timeout(time_to_wait, TooLong):
    gevent.sleep(10)

```

### 猴子补丁

现在这是gevent里面的一个难点。下面一个例子里可能看到 `monkey.patch_socket()`` 能够在运行时里面修改基础库socket：

```python
import socket

print(socket.socket)

print
"After monkey patch"
from gevent import monkey

monkey.patch_socket()

print(socket.socket)

import select

print(select.select)
monkey.patch_select()
print
"After monkey patch"
print(select.select)
```

Python的运行时里面允许能够大部分的对象都是可以修改的，包括模块，类和方法。这通常是一个坏主意，
然而在极端的情况下，当有一个库需要加入一些Python基本的功能的时候，monkey patch就能派上用场了。
在上面的例子里，gevent能够改变基础库里的一些使用IO阻塞模型的库比如socket，ssl，threading等等并且把它们改成协程的执行方式。

### 事件

有时候我们还需要在多个greenlet直接进行通信，比如某些操作的同步。事件(event)是一个在Greenlet之间异步通信的形式。

```python
import gevent
from gevent.event import Event

'''
Illustrates the use of events
'''

evt = Event()


def setter():
    '''After 3 seconds, wake all threads waiting on the value of evt'''
    print('A: Hey wait for me, I have to do something')
    gevent.sleep(3)
    print("Ok, I'm done")
    evt.set()


def waiter():
    '''After 3 seconds the get call will unblock'''
    print("I'll wait for you")
    evt.wait()  # blocking
    print("It's about time")


def main():
    gevent.joinall([
        gevent.spawn(setter),
        gevent.spawn(waiter),
        gevent.spawn(waiter),
        gevent.spawn(waiter),
    ])


if __name__ == '__main__':
    main()

```

事件对象的一个扩展是AsyncResult，它允许你在唤醒调用上附加一个值。
它有时也被称作是future或defered，因为它持有一个指向将来任意时间可设置为任何值的引用。

```python
import gevent
from gevent.event import AsyncResult

a = AsyncResult()


def setter():
    """
    After 3 seconds set the result of a.
    """
    gevent.sleep(3)
    a.set('Hello!')


def waiter():
    """
    After 3 seconds the get call will unblock after the setter
    puts a value into the AsyncResult.
    """
    print(a.get())


gevent.joinall([
    gevent.spawn(setter),
    gevent.spawn(waiter),
])
```

### 队列

队列是一个排序的数据集合，它有常见的put / get操作，但是它是以在Greenlet之间可以安全操作的方式来实现的。

举例来说，如果一个Greenlet从队列中取出一项，此项就不会被同时执行的其它Greenlet再取到了。

```python
import gevent
from gevent.queue import Queue

tasks = Queue()


def worker(n):
    while not tasks.empty():
        task = tasks.get()
        print('Worker %s got task %s' % (n, task))
        gevent.sleep(0)

    print('Quitting time!')


def boss():
    for i in range(1, 25):
        tasks.put_nowait(i)


gevent.spawn(boss).join()

gevent.joinall([
    gevent.spawn(worker, 'steve'),
    gevent.spawn(worker, 'john'),
    gevent.spawn(worker, 'nancy'),
])
```

如果需要，队列也可以阻塞在put或get操作上。

put和get操作都有非阻塞的版本，put_nowait和get_nowait不会阻塞，
然而在操作不能完成时抛出`gevent.queue.Empty`或`gevent.queue.Full`异常。

我们让boss与多个worker同时运行，并限制了queue不能放入多于3个元素：

```python
import gevent
from gevent.queue import Queue, Empty

tasks = Queue(maxsize=3)


def worker(n):
    try:
        while True:
            task = tasks.get(timeout=1)  # decrements queue size by 1
            print('Worker %s got task %s' % (n, task))
            gevent.sleep(0)
    except Empty:
        print('Quitting time!')


def boss():
    """
    Boss will wait to hand out work until a individual worker is
    free since the maxsize of the task queue is 3.
    """

    for i in xrange(1, 10):
        tasks.put(i)
    print('Assigned all work in iteration 1')

    for i in xrange(10, 20):
        tasks.put(i)
    print('Assigned all work in iteration 2')


gevent.joinall([
    gevent.spawn(boss),
    gevent.spawn(worker, 'steve'),
    gevent.spawn(worker, 'john'),
    gevent.spawn(worker, 'bob'),
])
```

### 组和池

组(group)是一个运行中greenlet的集合，集合中的greenlet像一个组一样会被共同管理和调度。
它也兼饰了像Python的multiprocessing库那样的平行调度器的角色:

```python
import gevent
from gevent.pool import Group


def talk(msg):
    for i in xrange(3):
        print(msg)


g1 = gevent.spawn(talk, 'bar')
g2 = gevent.spawn(talk, 'foo')
g3 = gevent.spawn(talk, 'fizz')

group = Group()
group.add(g1)
group.add(g2)
group.join()

group.add(g3)
group.join()
```

池(pool)是一个为处理数量变化并且需要限制并发的greenlet而设计的结构。
在需要并行地做很多受限于网络和IO的任务时常常需要用到它。

当构造gevent驱动的服务时，经常会将围绕一个池结构的整个服务作为中心。一个例子就是在各个socket上轮询的类。

```python
from gevent.pool import Pool


class SocketPool(object):

    def __init__(self):
        self.pool = Pool(1000)
        self.pool.start()

    def listen(self, socket):
        while True:
            socket.recv()

    def add_handler(self, socket):
        if self.pool.full():
            raise Exception("At maximum pool size")
        else:
            self.pool.spawn(self.listen, socket)

    def shutdown(self):
        self.pool.kill()
```

### 锁和信号量

信号量是一个允许greenlet相互合作，限制并发访问或运行的低层次的同步原语。信号量有两个方法，acquire和release。
在信号量是否已经被acquire或release，和拥有资源的数量之间不同，被称为此信号量的范围 (the bound of the semaphore)。
如果一个信号量的范围已经降低到0，它会阻塞acquire操作直到另一个已经获得信号量的greenlet作出释放。

```python
from gevent import sleep
from gevent.pool import Pool
from gevent.coros import BoundedSemaphore

sem = BoundedSemaphore(2)


def worker1(n):
    sem.acquire()
    print('Worker %i acquired semaphore' % n)
    sleep(0)
    sem.release()
    print('Worker %i released semaphore' % n)


def worker2(n):
    with sem:
        print('Worker %i acquired semaphore' % n)
        sleep(0)
    print('Worker %i released semaphore' % n)


pool = Pool()
pool.map(worker1, xrange(0, 2))
pool.map(worker2, xrange(3, 6))
```

范围为1的信号量也称为锁(lock)，它向单个greenlet提供了互斥访问。信号量和锁常常用来保证资源只在程序上下文被单次使用。

### 子进程

自gevent 1.0起，`gevent.subprocess`，一个Python subprocess模块的修补版本已经添加，它支持协作式的等待子进程。

```python
import gevent
from gevent.subprocess import Popen, PIPE


def cron():
    while True:
        print("cron")
        gevent.sleep(0.2)


g = gevent.spawn(cron)
sub = Popen(['sleep 1; uname'], stdout=PIPE, shell=True)
out, err = sub.communicate()
g.kill()
print(out.rstrip())
```

### actor模型

actor模型是一个由于Erlang变得普及的更高层的并发模型。简单的说它的主要思想就是许多个独立的Actor，
每个Actor有一个可以从其它Actor接收消息的收件箱。Actor内部的主循环遍历它收到的消息，并根据它期望的行为来采取行动。

Gevent没有原生的Actor类型，但在一个子类化的Greenlet内使用队列，我们可以定义一个非常简单的。

```python
import gevent
from gevent.queue import Queue


class Actor(gevent.Greenlet):

    def __init__(self):
        self.inbox = Queue()
        Greenlet.__init__(self)

    def receive(self, message):
        """
        Define in your subclass.
        """
        raise NotImplemented()

    def _run(self):
        self.running = True

        while self.running:
            message = self.inbox.get()
            self.receive(message)
```

接下来使用这个actor的例子：

```python
import gevent
from gevent.queue import Queue
from gevent import Greenlet


class Pinger(Actor):
    def receive(self, message):
        print(message)
        pong.inbox.put('ping')
        gevent.sleep(0)


class Ponger(Actor):
    def receive(self, message):
        print(message)
        ping.inbox.put('pong')
        gevent.sleep(0)


ping = Pinger()
pong = Ponger()

ping.start()
pong.start()

ping.inbox.put('start')
gevent.joinall([ping, pong])
```

