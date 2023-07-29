---
title: RabbitMQ简易教程 - 主题
date: 2017-05-15 16:33:17 +0800
toc: true
categories: [ 中间件 ]
tags: [ rabbitmq ]
---

前面一篇通过使用`direct`类型的交换机代替`fanout`广播类型交换机，
实现了一个基于日志级别路由对应的消息的功能。

但是还是有它的局限性——它并不能根据多个条件来实现路由，只能通过完全匹配`routing key`，
灵活性不够。

比如我想实现仅仅对于那些`error`级别日志并且由`kern`生成的日志才记录到文件中。
<!-- more -->

## 主题交换机

主题交换机的`binding key`和发送到该交换机的消息所带的`routing key`并不是一个简单的单词，
而是以点`.`隔开的单词序列。比如`stock.usd.nyse`、`nyse.vmw`等。

另外，有两个特殊的情况：

* *(星号)可代表任一个单词
* #（井字符）可代表0个或多个单词

比如有如下的主题交换机：

![](https://xnstatic-1253397658.file.myqcloud.com/rb07.png)

* `quick.orange.rabbit`会同时发送给2个队列
* `lazy.orange.elephant`会同时发送给2个队列
* `quick.orange.fox`仅仅发给第1个队列
* `lazy.brown.fox`仅仅发送给第2个队列
* `quick.brown.fox`被丢弃
* `quick.orange.male.rabbit`被丢弃
* `lazy.orange.male.rabbit`仅仅发送给第2个队列

## 最终代码

我们假设所有的消息的`routing key`形式为`"<facility>.<severity>`。

日志生产者：

```python emit_log_topic.py
import pika
import sys

connection = pika.BlockingConnection(pika.ConnectionParameters(host='192.168.217.161', port=5673))
channel = connection.channel()

channel.exchange_declare(exchange='topic_logs',
                         type='topic')

routing_key = 'disk.error'
message = '[disk.info] Hello World!'
channel.basic_publish(exchange='topic_logs',
                      routing_key=routing_key,
                      body=message)
print(" [x] Sent %r:%r" % (routing_key, message))

routing_key = 'disk.warning'
message = '[disk.warning] Hello World!'
channel.basic_publish(exchange='topic_logs',
                      routing_key=routing_key,
                      body=message)
print(" [x] Sent %r:%r" % (routing_key, message))


routing_key = 'test.error'
message = '[test.error] Hello World!'
channel.basic_publish(exchange='topic_logs',
                      routing_key=routing_key,
                      body=message)
print(" [x] Sent %r:%r" % (routing_key, message))


connection.close()
```

日志消费者：

```python receive_logs_topic.py
import pika
import sys

connection = pika.BlockingConnection(pika.ConnectionParameters(host='192.168.217.161', port=5673))
channel = connection.channel()

channel.exchange_declare(exchange='topic_logs',
                         type='topic')

result = channel.queue_declare(exclusive=True)
queue_name = result.method.queue

binding_keys = ['disk.error', 'disk.warning']
if not binding_keys:
    sys.stderr.write("Usage: %s [info] [warning] [error]\n" % sys.argv[0])
    sys.exit(1)

for binding_key in binding_keys:
    channel.queue_bind(exchange='direct_logs',
                       queue=queue_name,
                       routing_key=binding_key)

print(' [*] Waiting for logs. To exit press CTRL+C')


def callback(ch, method, properties, body):
    print(" [x] %r:%r" % (method.routing_key, body))


channel.basic_consume(callback,
                      queue=queue_name,
                      no_ack=True)

channel.start_consuming()
```

运行效果

emit_log_topic.py

```
 [x] Sent 'disk.error':'[disk.info] Hello World!'
 [x] Sent 'disk.warning':'[disk.warning] Hello World!'
 [x] Sent 'test.error':'[test.error] Hello World!'
```

receive_logs_topic.py

```
 [*] Waiting for logs. To exit press CTRL+C
 [x] 'disk.error':b'[disk.info] Hello World!'
 [x] 'disk.warning':b'[disk.warning] Hello World!'
```

可以看到，`test.error`的日志并没有被消费者拿到。
