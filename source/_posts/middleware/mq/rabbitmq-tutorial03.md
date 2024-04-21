---
title: RabbitMQ简易教程 - 发布订阅
date: 2017-05-10 13:14:22 +0800
toc: true
categories: [ 中间件 ]
tags: [ rabbitmq ]
---

前面一篇介绍的任务队列是每个消息只能被一个工作者取走。这一篇讲解发布/订阅消息模式，
在这个模式里面，一个消息可以被发送给多个消费者。

这里我通过一个简单的日志系统来说明，消息生产者会将日志发送给队列，然后多个订阅者可以接收到这条日志显示到不同的地方，
比如可以打印到文件中，同时打印到控制台上面。
<!-- more -->

## Exchanges

前面都是直接向某个队列发送/接受消息，现在是时候构建一个RabbitMQ中完整的消息模型了。

在RabbitMQ中，消息生产者从来不会将消息直接发送给某个队列，
甚至都不知道这个消息会不会发送到队列中去。

消息生产者将消息发送给交换机(exchange)，它负责接收生产者的消息，并将消息发送给多个队列。
交换机必须知道处理这个消息的规则，比如发送给某个特定队列，发送给多个队列，或者丢弃它。
这些规则是通过`exchange type`来定义。

![](https://xnstatic-1253397658.file.myqcloud.com/rb02.png)

有很多种交换机类型：direct、topic, headers 和 fanout。
这篇重点关注最后一个`fanout`类型。声明一个交换机：

```python
channel.exchange_declare(exchange='logs',
                         type='fanout')
```

fanout类型的将消息广播给它所知道的所有队列。

列出所有的交换机命令：

```bash
sudo rabbitmqctl list_exchanges
``

你会发现里面有很多`amq.*`开头的和一个空名字的默认交换机，这些都是系统自带的。

现在使用我们前面自己创建的交换机：
```python
channel.basic_publish(exchange='logs',
                      routing_key='',
                      body=message)
```

## 临时队列

我们需要每次连接至mq的时候使用一个新队列，使用完了就销毁，这里可使用临时队列：

```python
result = channel.queue_declare()
```

然后就可以通过`result.method.queue`获取临时队列名称，提供给消费者使用。
另外消费者用完后需要销毁，可添加一个`exclusive`选项：

```python
result = channel.queue_declare(exclusive=True)
```

## 绑定

交换机和队列直接的关系我们称为绑定`binding`

![](https://xnstatic-1253397658.file.myqcloud.com/rb03.png)

```python
channel.queue_bind(exchange='logs',
                   queue=result.method.queue)
```

这时候，`logs`交换机就会将它接收到的消息发送给我们的临时队列了。

查看系统所有的绑定命令：

```bash
rabbitmqctl list_binding
```

## 最终代码

最后我们开始把所有代码整理出来，消息生产者发送日志消息到`logs`交换机，
必须给这个交换机一个`routing_key`，但是对于fanout类型的会忽略这个值。

![](https://xnstatic-1253397658.file.myqcloud.com/rb04.png)

```python emit_log.py
#!/usr/bin/env python
import pika
import sys

connection = pika.BlockingConnection(pika.ConnectionParameters(host='192.168.217.161', port=5673))
channel = connection.channel()

channel.exchange_declare(exchange='logs',
                         type='fanout')

message = ' '.join(sys.argv[1:]) or "info: Hello World!"
channel.basic_publish(exchange='logs',
                      routing_key='',
                      body=message)
print(" [x] Sent %r" % message)
connection.close()

```

日志接受者实现：

```python receive_logs.py
import pika

connection = pika.BlockingConnection(pika.ConnectionParameters(host='192.168.217.161', port=5673))
channel = connection.channel()

channel.exchange_declare(exchange='logs',
                         type='fanout')

result = channel.queue_declare(exclusive=True)
queue_name = result.method.queue

channel.queue_bind(exchange='logs',
                   queue=queue_name)

print(' [*] Waiting for logs. To exit press CTRL+C')

def callback(ch, method, properties, body):
    print(" [x] %r" % body)

channel.basic_consume(callback,
                      queue=queue_name,
                      no_ack=True)

channel.start_consuming()
```

一切搞定，先运行接受者程序，如果你想把消息写入文件中，可以这样执行：

```bash
python receive_logs.py > test.log
```

如果只想打印到控制台：

```bash
python receive_logs.py
```

然后开始启动日志发送者进程：

```bash
python emit_log.py
```

演示结果：

第一个窗口

```bash
rabbitmqctl list_bindings
```

查看test.log内容

第二个窗口：

```bash
$ python receive_logs.py
 [*] Waiting for logs. To exit press CTRL+C
 [x] b'info: Hello World!'
```

发送者窗口：

```bash
$ python emit_log.py
 [x] Sent 'info: Hello World!'
```

最后，运行如下命令来确认绑定的确建立了：

```bash
[root@controller161 ~]# rabbitmqctl list_bindings
Listing bindings ...
	exchange	amq.gen-4CuguDMUeYD7j9rFJQvF9Q	queue	amq.gen-4CuguDMUeYD7j9rFJQvF9Q	[]
	exchange	amq.gen-TlEpq9mpE7zlfa3Qq6qFMA	queue	amq.gen-TlEpq9mpE7zlfa3Qq6qFMA	[]
	exchange	hello	queue	hello	[]
	exchange	task_queue	queue	task_queue	[]
logs	exchange	amq.gen-4CuguDMUeYD7j9rFJQvF9Q	queue	amq.gen-4CuguDMUeYD7j9rFJQvF9Q	[]
logs	exchange	amq.gen-TlEpq9mpE7zlfa3Qq6qFMA	queue	amq.gen-TlEpq9mpE7zlfa3Qq6qFMA	[]
```

很清楚，交换机`logs`里面的消息被发送给两个由服务器生成的名字的队列中去了，这正是我们想要的。

