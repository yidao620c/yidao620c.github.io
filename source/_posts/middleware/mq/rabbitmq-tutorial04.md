---
title: RabbitMQ简易教程 - 路由
date: 2017-05-13 19:17:55 +0800
toc: true
categories: [ 中间件 ]
tags: [ rabbitmq ]
---

前面一篇实现了一个非常基础的日志系统，交换机将所有接收到的消息广播到它所知道的多个接受者。

这一节我们更进一步，实现订阅部分消息的功能。比如我只讲那些ERROR级别的消息写入日志文件，
同时将所有日志打印到控制台上面。
<!-- more -->

## 绑定

在前面的例子我们已经创建了一个绑定：

```python
channel.queue_bind(exchange=exchange_name,
                   queue=queue_name)
```

简单讲就是这个队列对这个交换机上面的消息很感兴趣。

绑定需要指定一个关键字参数叫`routing_key`，为了不跟`basic_publish`搞混，
我现在将其称为`binding key`，下面我们创建一个带key的绑定：

```python
channel.queue_bind(exchange=exchange_name,
                   queue=queue_name,
                   routing_key='black')
```

绑定key的含义根据不同类型的交换机而不同，对于我前面使用的fanout类型的交换机，会忽略这个参数。

## Direct exchange

我现在需要根据日志级别来决定是否将消息写入文件，我只讲那些严重错误的日志写入文件，以节省磁盘空间。
那么我就不能再使用前面的fanout类型的交换机了，因为它只管广播消息到所有队列，其他不管。

我们先使用`direct`交换机，它的规则很简单，就是`binding key`完全一致就发送。

![](https://xnstatic-1253397658.file.myqcloud.com/rb05.png)

比如有上图这样的绑定关系，x交换机跟两个队列绑定，第一个队列使用`orange`的绑定关键字，
第二个队列使用了2个绑定关键字`black`和`green`。

这种情况下，如果某个消息发布到该交换机时候带的`routing key`值为`orange`的时候，这个消息会被路由到第一个队列，
如果某个消息发布到该交换机时候带的`routing key`值为`black`或`green`的时候，这个消息会被路由到第二个队列。
其他情况的消息都会被丢弃掉。

同时还能支持交换机使用同一个`binding key`来和多个队列绑定，
这时候如果某个消息的`routing key`一致，那么这个消息会同时发送给这几个队列。

## 最终代码

交换机和队列绑定图：

![](https://xnstatic-1253397658.file.myqcloud.com/rb06.png)

日志消息生产者代码：

```python emit_log_direct.py
import pika
import sys

connection = pika.BlockingConnection(pika.ConnectionParameters(host='192.168.217.161', port=5673))
channel = connection.channel()

channel.exchange_declare(exchange='direct_logs',
                         type='direct')

severity = 'info'
message = '[INFO] Hello World!'
channel.basic_publish(exchange='direct_logs',
                      routing_key=severity,
                      body=message)
print(" [x] Sent %r:%r" % (severity, message))

severity = 'error'
message = '[ERROR] Hello World!'
channel.basic_publish(exchange='direct_logs',
                      routing_key=severity,
                      body=message)
print(" [x] Sent %r:%r" % (severity, message))

severity = 'warning'
message = '[WARNING] Hello World!'
channel.basic_publish(exchange='direct_logs',
                      routing_key=severity,
                      body=message)
print(" [x] Sent %r:%r" % (severity, message))

connection.close()
```

我发送了三条消息，级别分别为`info`, `error` 和 `warning`。

日志消费者：

```python receive_logs_direct.py
import pika
import sys

connection = pika.BlockingConnection(pika.ConnectionParameters(host='192.168.217.161', port=5673))
channel = connection.channel()

channel.exchange_declare(exchange='direct_logs',
                         type='direct')

result = channel.queue_declare(exclusive=True)
queue_name = result.method.queue

severities = ['warning', 'error']

for severity in severities:
    channel.queue_bind(exchange='direct_logs',
                       queue=queue_name,
                       routing_key=severity)

print(' [*] Waiting for logs. To exit press CTRL+C')


def callback(ch, method, properties, body):
    print(" [x] %r:%r" % (method.routing_key, body))


channel.basic_consume(callback,
                      queue=queue_name,
                      no_ack=True)

channel.start_consuming()
```

我创建了一个临时队列，并将其和一个交换机绑定，并且指定`binding key`为`error`和`warning`。

运行效果：

日志生产者：

```
 [x] Sent 'info':'[INFO] Hello World!'
 [x] Sent 'error':'[ERROR] Hello World!'
 [x] Sent 'warning':'[WARNING] Hello World!'
```

日志消费者：

```
 [*] Waiting for logs. To exit press CTRL+C
 [x] 'error':b'[ERROR] Hello World!'
 [x] 'warning':b'[WARNING] Hello World!'
```

很明显，只有`error`和`warning`级别的日志被消费了。

