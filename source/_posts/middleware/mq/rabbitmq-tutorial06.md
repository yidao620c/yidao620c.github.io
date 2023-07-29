---
title: RabbitMQ简易教程 - RPC
date: 2017-05-16 09:09:37 +0800
toc: true
categories: [ 中间件 ]
tags: [ rabbitmq ]
---

在教程第二篇里面我们学习了如何实现一个任务队列，异步方式去处理那些比较耗时的任务。
但是如果我们需要调用一个远程主机上面的方法，并且等待它的执行结果呢？
这种模式我们通常将它称为远程方法调用（RPC）。

这一篇我们将利用RabbitMQ来构建一个RPC服务，服务器上面有一个可返回斐波拉契数的函数。
客户端通过rpc调用来获取结果。
<!-- more -->

## 客户端接口

为了演示的方便，我在客户端创建一个简单的类，里面有个`call`方法，它会发送一个rpc请求并等待执行的返回结果。

使用它的方式如下：

```python
fibonacci_rpc = FibonacciRpcClient()
result = fibonacci_rpc.call(4)
print("fib(4) is %r" % result)
```

## 回调队列

使用RabbitMQ实现RPC原理很简单，客户端使用消息的方式发送方法名和参数，服务器将结果也作为消息回传回来，
那么这时候就需要另外一个队列来接受返回的消息，这里可指定回调队列。

```python
result = channel.queue_declare(exclusive=True)
callback_queue = result.method.queue

channel.basic_publish(exchange='',
                      routing_key='rpc_queue',
                      properties=pika.BasicProperties(
                            reply_to = callback_queue,
                            ),
                      body=request)

# ... and some code to read a response message from the callback_queue ...
```

## Correlation id

上面的例子每一次的RPC请求都会创建一个新的回调队列，这个很浪费资源。
不过我们现在可以通过`Correlation id`，也就是关联id来对每个客户端只创建唯一的回调队列。
每次请求带上这个关联id，当获取返回值时候比较关联id，一致的说明是这个请求的返回消息就接受，不一致的忽略这个消息。

## Summary

一次RPC调用流程大致如下：

![](https://xnstatic-1253397658.file.myqcloud.com/rb08.png)

1. 客户端启动时候，创建一个匿名的排他回调队列
2. 每一次的RPC请求，客户端发送的消息都会带两个属性：`reply_to`(回调队列)和correlation_id
3. 请求发送至`rpc_queue`队列
4. RPC服务器订阅这个`rpc_queue `队列，收到消息就执行然后返回结果，使用`reply_to`指定的回调队列。
5. 客户端等待回调队列的数据返回，当收到消息后，检查`correlation_id`属性，如果一致就返回结果。

## 最终代码

RPC服务器：

```python rpc_server.py
import pika

connection = pika.BlockingConnection(pika.ConnectionParameters(host='192.168.217.161', port=5673))

channel = connection.channel()

channel.queue_declare(queue='rpc_queue')


def fib(n):
    if n == 0:
        return 0
    elif n == 1:
        return 1
    else:
        return fib(n - 1) + fib(n - 2)


def on_request(ch, method, props, body):
    n = int(body)

    print(" [.] fib(%s)" % n)
    response = fib(n)

    ch.basic_publish(exchange='',
                     routing_key=props.reply_to,
                     properties=pika.BasicProperties(correlation_id=props.correlation_id),
                     body=str(response))
    ch.basic_ack(delivery_tag=method.delivery_tag)


channel.basic_qos(prefetch_count=1)
channel.basic_consume(on_request, queue='rpc_queue')

print(" [x] Awaiting RPC requests")
channel.start_consuming()
```

客户端调用：

```python rpc_client.py
import pika
import uuid


class FibonacciRpcClient(object):
    def __init__(self):
        self.connection = pika.BlockingConnection(pika.ConnectionParameters(host='192.168.217.161', port=5673))

        self.channel = self.connection.channel()

        result = self.channel.queue_declare(exclusive=True)
        self.callback_queue = result.method.queue

        self.channel.basic_consume(self.on_response, no_ack=True,
                                   queue=self.callback_queue)

    def on_response(self, ch, method, props, body):
        if self.corr_id == props.correlation_id:
            self.response = body

    def call(self, n):
        self.response = None
        self.corr_id = str(uuid.uuid4())
        self.channel.basic_publish(exchange='',
                                   routing_key='rpc_queue',
                                   properties=pika.BasicProperties(
                                       reply_to=self.callback_queue,
                                       correlation_id=self.corr_id,
                                   ),
                                   body=str(n))
        while self.response is None:
            self.connection.process_data_events()
        return int(self.response)


fibonacci_rpc = FibonacciRpcClient()

print(" [x] Requesting fib(30)")
response = fibonacci_rpc.call(30)
print(" [.] Got %r" % response)
```

演示效果

rpc_server.py

```
 [x] Awaiting RPC requests
 [.] fib(30)
```

rpc_client.py

```
 [x] Requesting fib(30)
 [.] Got 832040
```

