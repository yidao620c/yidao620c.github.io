---
title: python核心 - 并发编程
date: 2015-12-18 22:22:22 +0800
toc: true
categories: [ python ]
tags: [ python核心 ]
---

现在是多核和并发时代，所以不管什么语言都要支持这个特性。并发是看上去同时执行，并行是在多核上同时执行。

我们先解释下线程和进程。简单来说，一个任务就是一个进程（Process）。
在一个进程内部，要同时干多件事，就需要同时运行多个进程内的"子任务"，这些子任务就叫线程（Thread）

线程是最小的执行单元，而进程由至少一个线程组成。如何调度进程和线程，
完全由操作系统决定，程序自己不能决定什么时候执行，执行多长时间。

多进程和多线程的程序涉及到同步、数据共享的问题，编写起来更复杂。
<!-- more -->

python中实现并发方式有下面几种：

1. 多进程模式
2. 多线程模式
3. 多进程+多线程模式
4. 协程模式

### fork函数

我们先要讲一下fork()函数，这个函数只在Unix/Linux/Max上有效，在Windows系统上无效。

一般的函数调用都是一次调用一次返回，而这个特殊的fork()函数调用一次返回两次。
子进程永远返回0，而父进程返回子进程ID

```python
import os

print('Process (%s) start...' % os.getpid())
pid = os.fork()
if pid == 0:
    print('I am child process (%s) and my parent is %s.' % (os.getpid(), os.getppid()))
else:
    print('I (%s) just created a child process (%s).' % (os.getpid(), pid))

```

### threading

编写跨平台的并发程序就不能直接使用上面的fork函数了。

Python主要通过标准库中的threading包来实现多线程。并且Python的线程是真正的Posix Thread，而不是模拟出来的线程

启动一个线程就是把一个函数传入并创建Thread实例，然后调用start()开始执行，如果要同时修改全局共享变量，需要线程锁：

```python
import time, threading

# 假定这是你的银行存款:
balance = 0
lock = threading.Lock()


def change_it(n):
    # 先存后取，结果应该为0:
    global balance
    balance = balance + n
    balance = balance - n


def run_thread(n):
    for i in range(100000):
        # 先要获取锁:
        lock.acquire()
        try:
            # 放心地改吧:
            change_it(n)
        finally:
            # 改完了一定要释放锁:
            lock.release()


t1 = threading.Thread(target=run_thread, args=(5,))
t2 = threading.Thread(target=run_thread, args=(8,))
t1.start()
t2.start()
t1.join()
t2.join()
print(balance)
```

另外还可以通过OOP方式创建线程，其实就是定义一个类继承thread.Threading类，内部覆盖run()方法即可。

如果要管理多个线程，还可以使用线程池方式：

```python
from multiprocessing.dummy import Pool as ThreadPool
import time, random


def hello(str):
    time.sleep(2)
    print(str)
    return str


def print_result(request, result):
    print("the result is %s %r" % (request.requestID, result))


data = [random.randint(1, 10) for i in range(20)]
# Make the Pool of workers
pool = ThreadPool(6)
# Open the urls in their own threads
# and return the results
results = pool.map(hello, data)
# callback
results = pool.map_async(hello, data, chunksize=None, callback=callback)
# apply
result = pool.apply(hello, data[0])
result = pool.apply_async(hello, data[0], callback=callback)

# close the pool and wait for the work to finish
pool.close()
pool.join()
```

这里还得提一下Python中的GIL锁（Global Interpreter Lock），在多核机器上面，无论开启多少个线程同时执行，
只可能会占用一个CPU，因为GIL锁是CPython这个解释器定义的。所以多线程程序不要指望能利用多核，不过不用担心，我们可以用多进程方式。

### ThreadLocal

多线程下我们应该尽量使用局部变量，但是局部变量在函数多次调用的时候，传递起来很麻烦。

```python
def process_student(name):
    std = Student(name)
    # std是局部变量，但是每个函数都要用它，因此必须传进去：
    do_task_1(std)
    do_task_2(std)


def do_task_1(std):
    do_subtask_1(std)
    do_subtask_2(std)


def do_task_2(std):
    do_subtask_2(std)
    do_subtask_2(std)
```

每个函数一层一层调用都这么传参数那还得了？用全局变量？也不行，因为每个线程处理不同的Student对象，不能共享。

最好的方法是用一个全局dict存放所有的Student对象，然后以thread自身作为key获得线程对应的Student对象。
ThreadLocal应运而生，不用查找dict，ThreadLocal帮你自动做这件事：

```python
import threading

# 创建全局ThreadLocal对象:
local_school = threading.local()


def process_student():
    # 获取当前线程关联的student:
    std = local_school.student
    print('Hello, %s (in %s)' % (std, threading.current_thread().name))


def process_thread(name):
    # 绑定ThreadLocal的student:
    local_school.student = name
    process_student()


t1 = threading.Thread(target=process_thread, args=('Alice',), name='Thread-A')
t2 = threading.Thread(target=process_thread, args=('Bob',), name='Thread-B')
t1.start()
t2.start()
t1.join()
t2.join()
```

全局变量local_school就是一个ThreadLocal对象，每个Thread对它都可以读写student属性，但互不影响。
你可以把local_school看成全局变量，但每个属性如local_school.student都是线程的局部变量，
可以任意读写而互不干扰，也不用管理锁的问题，ThreadLocal内部会处理。

ThreadLocal最常用的地方就是为每个线程绑定一个数据库连接，HTTP请求，用户身份信息等，
这样一个线程的所有调用到的处理函数都可以非常方便地访问这些资源。

### subprocess

一个进程可以fork一个子进程，并让这个子进程exec另外一个程序。要编写扩平台的多进程程序，就不能直接使用fork。
在Python中，我们通过标准库中的subprocess包来fork一个子进程，并运行一个外部的程序。

```python
# 执行基本系统命令
ret = subprocess.call('ls -l', shell=True)

# 静默执行基本系统命令
ret = subprocess.call('rf -f *.java', shell=True,
                      stdout=open('/dev/null'))

# 执行命令，但是捕捉输出
p = subprocess.Popen('ls -l', shell=True,
                     stdout=subprocess.PIPE)
out = p.stdout.read()

# 执行命令，但是发送输入和接受输出
p = subprocess.Popen('wc', shell=True, stdin=subprocess.PIPE,
                     stdout=subprocess.PIPE, stderr=subprocess.PIPE)
out, err = p.communicate('.')

# 创建两个子进程，通过管道通信，实现命令"ls -l | wc"
p1 = subprocess.Popen('ls -l', stdout=subprocess.PIPE)
p2 = subprocess.Popen('wc', stdin=p1.stdout, stdout=subprocess.PIPE)
p1.stdout.close()  # Allow p1 to receive a SIGPIPE if p2 exits.
stdout, stderr = p2.communicate()
return p2.returncode, stdout, stderr
```

通过使用subprocess包，我们可以运行外部程序。这极大的拓展了Python的功能。
如果你已经了解了操作系统的某些应用，你可以从Python中直接调用该应用(而不是完全依赖Python)，
并将应用的结果输出给Python，并让Python继续处理。shell的功能(比如利用文本流连接各个应用)，就可以在Python中实现。

### multiprocessing

使用subprocess包来创建子进程，有两个很大的局限性：

1. 我们只能让subprocess运行外部的程序，而不是运行一个Python脚本内部编写的函数
2. 进程间只通过管道进行通信

以上限制了我们将subprocess包应用到更广泛的多进程任务。
这样的比较实际是不公平的，因为subprocessing本身就是设计成为一个shell，而不是一个多进程管理包。

`multiprocessing`包是Python中的多进程管理包。它可以利用`multiprocessing.Process`对象来创建一个进程，
该进程可以运行在Python程序内部编写的函数。该Process对象与Thread对象的用法相同，也有`start()`, `run()`, `join()`的方法。
此外`multiprocessing`包中也有`Lock/Event/Semaphore/Condition`类
(这些对象可以像多线程那样，通过参数传递给各个进程)，用以同步进程，其用法与threading包中的同名类一致。
所以，multiprocessing的很大一部份与threading使用同一套API，只不过换到了多进程的情境。

使用multiprocessing的API的时候，有几点需要注意：

1. 在UNIX平台上，当某个进程终结之后，该进程需要被其父进程调用wait，否则进程成为僵尸进程(Zombie)。
   所以，有必要对每个Process对象调用join()方法 (实际上等同于wait)。对于多线程来说，由于只有一个进程，所以不存在此必要性。
2. `multiprocessing`提供了threading包中没有的IPC(比如Pipe和Queue)，效率上更高。应优先考虑Pipe和Queue
3. 多进程应该避免共享资源，比如全局变量或者传递参数

下面演示下它的基本用法：

```python
from multiprocessing import Process
import os


# 子进程要执行的代码
def run_proc(name):
    print('Run child process %s (%s)...' % (name, os.getpid()))


if __name__ == '__main__':
    print('Parent process %s.' % os.getpid())
    p = Process(target=run_proc, args=('test',))
    print('Process will start.')
    p.start()
    # join()方法可以等待子进程结束后再继续往下运行，通常用于进程间的同步。
    p.join()
    print('Process end.')
```

如果要启动大量的子进程，可以用进程池的方式批量创建子进程：

```python
from multiprocessing import Pool
import os, time, random


def long_time_task(name):
    print('Run task %s (%s)...' % (name, os.getpid()))
    start = time.time()
    time.sleep(random.random() * 3)
    end = time.time()
    print('Task %s runs %0.2f seconds.' % (name, (end - start)))


if __name__ == '__main__':
    print('Parent process %s.' % os.getpid())
    # Pool默认的大小时CUP核心数，这个是有意这样设计的
    p = Pool()
    for i in range(5):
        p.apply_async(long_time_task, args=(i,))
    print('Waiting for all subprocesses done...')
    # 对Pool对象调用join()方法会等待所有子进程执行完毕，
    # 调用join()之前必须先调用close()，调用close()之后就不能继续添加新的Process了。
    p.close()
    p.join()
    print('All subprocesses done.')
```

对Pool对象调用join()方法会等待所有子进程执行完毕，调用join()之前必须先调用close()，
调用close()之后就不能继续添加新的Process了

### 进程间通信

multiprocessing包中有Pipe类和Queue类来分别支持管道和消息队列这两种IPC机制。

#### Pipe方式

Pipe可以是单向(half-duplex)，也可以是双向(duplex)。mutiprocessing.Pipe(duplex=False)创建单向管道，
一个进程从PIPE一端输入对象，然后被PIPE另一端的进程接收。

```python
import multiprocessing as mul


def proc1(pipe):
    pipe.send('hello')
    print('proc1 rec:', pipe.recv())


def proc2(pipe):
    print('proc2 rec:', pipe.recv())
    pipe.send('hello, too')


# Build a pipe
pipe = mul.Pipe()
# Pass an end of the pipe to process 1
p1 = mul.Process(target=proc1, args=(pipe[0],))
# Pass the other end of the pipe to process 2
p2 = mul.Process(target=proc2, args=(pipe[1],))
p1.start()
p2.start()
p1.join()
p2.join()
```

这里的Pipe是双向的。

Pipe对象建立的时候，返回一个含有两个元素的表，每个元素代表Pipe的一端(Connection对象)。
我们对Pipe的某一端调用send()方法来传送对象，在另一端使用recv()来接收。

下面一个程序演示如何使用一个PIPE来获取`multiprocessing.Process`进程返回值，让人也可以通过Queue来达到同样效果。

```python
import multiprocessing


def worker(procnum, send_end):
    '''worker function'''
    result = str(procnum) + ' represent!'
    print(result)
    send_end.send(result)


def main():
    jobs = []
    pipe_list = []
    for i in range(5):
        # 单向管道返回的是(接受端，发送端)
        recv_end, send_end = multiprocessing.Pipe(False)
        p = multiprocessing.Process(target=worker, args=(i, send_end))
        jobs.append(p)
        pipe_list.append(recv_end)
        p.start()

    for proc in jobs:
        proc.join()
    result_list = [x.recv() for x in pipe_list]
    print(result_list)
    for x in pipe_list:
        x.close()


if __name__ == '__main__':
    main()
```

#### Queue方式

Queue与Pipe相类似，都是先进先出的结构。但Queue允许多个进程放入，多个进程从队列取出对象。
Queue使用mutiprocessing.Queue(maxsize)创建，maxsize表示队列中可以存放对象的最大数量。

```python
from multiprocessing import Process, Queue
import os, time, random


# 写数据进程执行的代码:
def write(q):
    for value in ['A', 'B', 'C']:
        print('Put %s to queue...' % value)
        q.put(value)
        time.sleep(random.random())


# 读数据进程执行的代码:
def read(q):
    while True:
        value = q.get(True)
        print('Get %s from queue.' % value)


if __name__ == '__main__':
    # 父进程创建Queue，并传给各个子进程：
    q = Queue()
    pw = Process(target=write, args=(q,))
    pr = Process(target=read, args=(q,))
    # 启动子进程pw，写入:
    pw.start()
    # 启动子进程pr，读取:
    pr.start()
    # 等待pw结束:
    pw.join()
    # pr进程里是死循环，无法等待其结束，只能强行终止:
    pr.terminate()
```

#### 共享变量方式

几个进程之间的都拥有自己独立的命名空间和地址空间，无法通过一些全局变量来实现，
`multiprocessing`提供了一些特殊的函数来实现共享变量：

**`Value`,`Array`的方式**

```python
from multiprocessing import Process, Value, Array


def f(n, a):
    n.value = 3.1415926
    for i in range(len(a)):
        a[i] = -a[i]


if __name__ == "__main__":
    num = Value('d', 0.0)
    arr = Array('i', range(10))
    p = Process(target=f, args=(num, arr))
    p.start()
    p.join()
    print
    num.value
    print
    arr[:]
```

`Value()`和`Array()`都有两个参数第一个参数代表存放的值的类型，第二个参数代表其值。

**Manager的方式**
这个方式支持的类型更多，灵活性更大，但是速度要慢于`Value`, `Array`

```python
from multiprocessing import Process, Manager


def f(d, l):
    d[1] = '1'
    d['2'] = 2
    d[0.25] = None
    l.reverse()


if __name__ == "__main__":
    manager = Manager()
    d = manager.dict()
    l = manager.list(range(10))
    p = Process(target=f, args=(d, l))
    p.start()
    p.join()
    print
    d
    print
    l
```

### 异步I/O

考虑到CPU和IO之间巨大的速度差异，一个任务在执行的过程中大部分时间都在等待IO操作，
单进程单线程模型会导致别的任务无法并行执行，因此，我们才需要多进程模型或者多线程模型来支持多任务并发执行。

现代操作系统对IO操作已经做了巨大的改进，最大的特点就是支持异步IO。如果充分利用操作系统提供的异步IO支持，
就可以用单进程单线程模型来执行多任务，这种全新的模型称为事件驱动模型，Nginx就是支持异步IO的Web服务器，
它在单核CPU上采用单进程模型就可以高效地支持多任务。在多核CPU上，可以运行多个进程（数量与CPU核心数相同），
充分利用多核CPU。由于系统总的进程数量十分有限，因此操作系统调度非常高效。用异步IO编程模型来实现多任务是一个主要的趋势。

对应到Python语言，单进程的异步编程模型称为协程，有了协程的支持，就可以基于事件驱动编写高效的多任务程序。
python语言通过yield关键字来支持协程，而第三方框架gevent则是一个非常优秀的协程框架。

### 并发方式总结

#### 线程（Thread）

```python
import threading


class worker(threading.Thread):
    def __init__(self):
        pass

    def run():
        """thread worker function"""
        print
        'Worker'


t = worker()
t.start()
```

#### 进程（Process）

```python
from multiprocessing import Process


def worker():
    """thread worker function"""
    print
    'Worker'


p = Process(target=worker)
p.start()
p.join()
```

由于线程共享相同的地址空间和内存，所以线程之间的通信是非常容易的，然而进程之间的通信就要复杂一些了。
常见的进程间通信有，管道，消息队列，Socket接口（TCP/IP）等等。

Python的mutliprocess模块提供了封装好的管道和队列，可以方便的在进程间传递消息。

Python进程间的同步使用锁，这一点喝线程是一样的。

另外，Python还提供了进程池Pool对象，可以方便的管理和控制线程。

#### 远程分布式主机 （Distributed Node）

随着大数据时代的到临，摩尔定理在单机上似乎已经失去了效果，数据的计算和处理需要分布式的计算机网络来运行，
程序并行的运行在多个主机节点上，已经是现在的软件架构所必需考虑的问题。

远程主机间的进程间通信有几种常见的方式:

* TCP／IP
* 远程方法调用 Remote Function Call，Python下有一个开源的实现[RPyC](http://rpyc.readthedocs.org/)
* 远程对象 Remote Object
* 消息队列 Message Queue

其中用的最多的还是消息队列，常见的支持Python接口的消息机制有：

* [RabbitMQ](http://www.rabbitmq.com/)
* [ZeroMQ](http://zguide.zeromq.org/)
* [Kafka](http://kafka.apache.org/)

#### 协程（Pseudo－Thread）

协程看上去像是线程，使用的接口类似线程接口，但是实际使用非线程的方式，对应的线程开销也不存的。

eventlet，gevent和concurence都是基于greenlet提供并发的。

gevent和eventlet类似:

```python
import gevent
from gevent import socket

urls = ['www.google.com', 'www.example.com', 'www.python.org']
jobs = [gevent.spawn(socket.gethostbyname, url) for url in urls]
gevent.joinall(jobs, timeout=2)

print[job.value
for job in jobs]
```

对于IO密集型的任务，使用多线程，或者是多进程都可以有效的提高程序的效率，
而使用伪线程性能提升非常显著，eventlet比没有并发的情况下，响应时间从9秒提高到0.03秒。
同时eventlet／gevent提供了非阻塞的异步调用模式，非常方便。
这里推荐使用线程或者协程，因为在响应时间类似的情况下，线程和协程消耗的资源更少。

