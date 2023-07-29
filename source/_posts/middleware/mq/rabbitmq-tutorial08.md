---
title: RabbitMQ简易教程 - 并发调度
date: 2017-05-26 09:17:21 +0800
toc: true
categories: [ 中间件 ]
tags: [ rabbitmq ]
---

RabbitMQ任务调度默认是阻塞的，使用pika中的`channel.start_consuming()`的时候，
每次收到一条消息后会顺序执行完回调函数，发送ACK的确认消息，然后再执行下一条消息。
虽然说可同时接受多条消息，但是并不能同时处理这多条消息，那么需要自己在代码里面实现任务的并发调度。

在Python里面实现并发方式多种多样，有多线程、多进程、多协程方式，我演示下如何实现。
<!-- more -->

## 默认方式

我先使用默认方式看看是否阻塞，消费者处理消息的回调函数中，我特定使用`time.sleep(5)`来模拟长时间任务。
设置`channel.basic_qos(prefetch_count=2)`可控制一次可接受几条消息，这里是2条。

生产者代码（后面的都不变），直接复制的前面教程的代码：

```python
import pika
import sys

connection = pika.BlockingConnection(
    pika.ConnectionParameters(host='192.168.217.161', port=5673))
channel = connection.channel()

channel.exchange_declare(exchange='topic_logs',
                         type='topic')

routing_key = 'disk.error'
message = '[disk.error] Hello World! 111111'
channel.basic_publish(exchange='topic_logs',
                      routing_key=routing_key,
                      body=message)

print(" [x] Sent %r:%r" % (routing_key, message))

routing_key = 'disk.error'
message = '[disk.error] Hello World! 222222'
channel.basic_publish(exchange='topic_logs',
                      routing_key=routing_key,
                      body=message)

print(" [x] Sent %r:%r" % (routing_key, message))

routing_key = 'disk.warning'
message = '[disk.warning] Hello World! 333333'
channel.basic_publish(exchange='topic_logs',
                      routing_key=routing_key,
                      body=message)
print(" [x] Sent %r:%r" % (routing_key, message))

connection.close()
```

消费者代码：

```python
import pika
import time
import datetime

connection = pika.BlockingConnection(pika.ConnectionParameters(host='192.168.217.161', port=5673))
channel = connection.channel()

channel.exchange_declare(exchange='topic_logs',
                         type='topic')

result = channel.queue_declare(exclusive=True)
queue_name = result.method.queue

binding_keys = ['disk.error', 'disk.warning']

for binding_key in binding_keys:
    channel.queue_bind(exchange='topic_logs',
                       queue=queue_name,
                       routing_key=binding_key)

print(' [*] Waiting for logs. To exit press CTRL+C')


def test(ch, method, body):
    print("%s [x] Received %r" % (datetime.datetime.now(), body,))
    time.sleep(5)
    print("%s [x] Finished %r" % (datetime.datetime.now(), body,))
    ch.basic_ack(delivery_tag=method.delivery_tag)


def callback(ch, method, properties, body):
    test(ch, method, body)


channel.basic_qos(prefetch_count=2)  # 并发的数量
channel.basic_consume(callback,
                      queue=queue_name)
channel.start_consuming()

```

运行结果：

```
 [*] Waiting for logs. To exit press CTRL+C
2017-05-26 11:04:46.977268 [x] Received b'[disk.error] Hello World! 111111'
2017-05-26 11:04:51.987554 [x] Finished b'[disk.error] Hello World! 111111'
2017-05-26 11:04:51.988555 [x] Received b'[disk.error] Hello World! 222222'
2017-05-26 11:04:56.988841 [x] Finished b'[disk.error] Hello World! 222222'
2017-05-26 11:04:56.989841 [x] Received b'[disk.warning] Hello World! 333333'
2017-05-26 11:05:02.001127 [x] Finished b'[disk.warning] Hello World! 333333'
```

可以看出，虽然我将消息接收设置成2，但是每次只能处理1条消息。

## 多线程方式

下面用多线程方式实现并发，消费者代码如下：

```python
import pika
import time
import datetime
import threading

connection = pika.BlockingConnection(pika.ConnectionParameters(host='192.168.217.161', port=5673))
channel = connection.channel()

channel.exchange_declare(exchange='topic_logs',
                         type='topic')

result = channel.queue_declare(exclusive=True)
queue_name = result.method.queue

binding_keys = ['disk.error', 'disk.warning']

for binding_key in binding_keys:
    channel.queue_bind(exchange='topic_logs',
                       queue=queue_name,
                       routing_key=binding_key)

print(' [*] Waiting for logs. To exit press CTRL+C')


def test(ch, method, body):
    print("%s [x] Received %r" % (datetime.datetime.now(), body,))
    time.sleep(5)
    print("%s [x] Finished %r" % (datetime.datetime.now(), body,))
    ch.basic_ack(delivery_tag=method.delivery_tag)


def callback(ch, method, properties, body):
    t = threading.Thread(target=test, args=(ch, method, body))
    t.start()


channel.basic_qos(prefetch_count=2)  # 并发的数量
channel.basic_consume(callback,
                      queue=queue_name)
channel.start_consuming()
```

运行结果：

```
 [*] Waiting for logs. To exit press CTRL+C
2017-05-26 11:07:58.486085 [x] Received b'[disk.error] Hello World! 111111'
2017-05-26 11:07:58.487085 [x] Received b'[disk.error] Hello World! 222222'
2017-05-26 11:08:03.486371 [x] Finished b'[disk.error] Hello World! 111111'
2017-05-26 11:08:03.487371 [x] Finished b'[disk.error] Hello World! 222222'
2017-05-26 11:08:03.489371 [x] Received b'[disk.warning] Hello World! 333333'
2017-05-26 11:08:08.489657 [x] Finished b'[disk.warning] Hello World! 333333'
```

很明显可以同时处理两条消息了。

## 多进程方式

下面用多进程方式实现并发（注意多进程只能在Linux中使用），消费者代码如下，我只写差异部分，其他跟上面多线程一样：

```python
from multiprocessing import Process


def callback(ch, method, properties, body):
    t = Process(target=test, args=(ch, method, body))
    t.start()
```

运行结果和多线程类似

## 协程方式

利用gevent协程模式，比多线程更加高效，这个是我推荐的方式：

```python
import gevent
from gevent import monkey;

monkey.patch_all()


def callback(ch, method, properties, body):
    # 协程启动，没有调用join，因为rabbitmq本身是阻塞的,可以不用join
    gevent.spawn(test, ch, method, body)
```

运行结果：

```
 [*] Waiting for logs. To exit press CTRL+C
2017-05-26 11:13:53.637398 [x] Received b'[disk.error] Hello World! 111111'
2017-05-26 11:13:53.637398 [x] Received b'[disk.error] Hello World! 222222'
2017-05-26 11:13:58.637684 [x] Finished b'[disk.error] Hello World! 222222'
2017-05-26 11:13:58.637684 [x] Finished b'[disk.error] Hello World! 111111'
2017-05-26 11:13:58.638684 [x] Received b'[disk.warning] Hello World! 333333'
2017-05-26 11:14:03.638970 [x] Finished b'[disk.warning] Hello World! 333333'
```

## Celery

对于简单的任务调度使用gevent就可以满足了，如果是大型复杂的系统需要任务调度，
可以考虑使用Celery这个框架。

Celery(芹菜)是一个异步任务队列/基于分布式消息传递的作业队列。它侧重于实时操作，但对调度支持也很好。
使用Python编写，但该协议可以在任何语言实现。它也可以与其他语言通过webhooks实现。

它的基本工作就是管理分配任务到不同的服务器，并且取得结果。至于说服务器之间是如何进行通信的？
这个Celery本身不能解决。所以，RabbitMQ作为一个消息队列管理工具被引入到和Celery集成，
负责处理服务器之间的通信任务。

现在的Celery早已增加了一些对Redis,MongoDB之类的支持。原因是RabbitMQ尽管足够强大，
但对于一些相对简单的业务环境来说可能太多（复杂）了一些。这样用户可以有多一些的选择。

官方集成RabbitMQ的文档：<http://docs.celeryproject.org/en/latest/getting-started/brokers/rabbitmq.html>

我这里就不讲怎样使用Celery了，后续有时间再弄个教程出来。

