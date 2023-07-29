---
title: RabbitMQ简易教程 - 任务队列
date: 2017-05-08 09:15:22 +0800
toc: true
categories: [ 中间件 ]
tags: [ rabbitmq ]
---

这里演示的官网通过python使用消息队列的教程：<https://www.rabbitmq.com/getstarted.html>

先演示最简单的一个入门级别的`hello world`例子，
发送者发送一个字符串，接受者接收到消息后打印出来。然后再介绍怎样实现任务队列
<!-- more -->

## hello world

这里使用 python 的pika来演示，先安装pika:

```bash
pip install pika
```

发送者程序需要先建立一个到RabbitMQ的连接(ip地址就是rabbitmq服务器地址)：

```python
#!/usr/bin/env python
import pika

connection = pika.BlockingConnection(pika.ConnectionParameters('192.168.217.161'))
channel = connection.channel()
```

然后创建一个消息队列：

```python
channel.queue_declare(queue='hello')
```

这里的`queue_declare`可以调用任意多次，因为最终只会有一个名字叫"hello"的队列被创建。

然后通过默认的路由器来转发消息：

```python
channel.basic_publish(exchange='',
                      routing_key='hello',
                      body='Hello World!')
print(" [x] Sent 'Hello World!'")
```

最后别忘了关闭连接来刷新缓存，确保所有的消息都写到了队列中：

```python
connection.close()
```

最后的接受者程序是这样的：

```python send.py
#!/usr/bin/env python
import pika

connection = pika.BlockingConnection(pika.ConnectionParameters(host='192.168.217.161', port=5673))
channel = connection.channel()

channel.queue_declare(queue='hello')

channel.basic_publish(exchange='',
                      routing_key='hello',
                      body='Hello World!')
print(" [x] Sent 'Hello World!'")
connection.close()
```

接受者程序差不多，不需要路由器，但是需要传递一个回调函数，每次接收到消息就被调用：

```python receive.py
#!/usr/bin/env python
import pika

connection = pika.BlockingConnection(pika.ConnectionParameters(host='192.168.217.161', port=5673))
channel = connection.channel()

channel.queue_declare(queue='hello')

def callback(ch, method, properties, body):
    print(" [x] Received %r" % body)

channel.basic_consume(callback,
                      queue='hello',
                      no_ack=True)

print(' [*] Waiting for messages. To exit press CTRL+C')
channel.start_consuming()
```

出现的问题：

```
pika.exceptions.ProbableAuthenticationError
```

查看日志：`/var/log/rabbitmq/deploy_rabbitmq.log`

发现里面的错误是这样的

```
=ERROR REPORT==== 23-May-2017::10:57:22 ===
Error on AMQP connection <0.460.0> (10.10.111.230:58575 -> 192.168.217.161:5673, state: starting):
PLAIN login refused: user 'guest' - invalid credentials
```

解决办法是，修改guest的密码：

```bash
[root@controller161 ~]# rabbitmqctl list_users
Listing users ...
guest	[administrator]
test	[administrator]
[root@controller161 ~]# rabbitmqctl change_password guest guest
Changing password for user "guest" ...
```

再次发送就没有问题了。

发送者发完消息就退出去了，而接受者会永久循环，测试完成通过"Ctrl+C"来结束进程。

## 任务队列

任务队列（或者叫工作队列）的思想是对于处理那些非常耗时的任务的时候，通过异步消息队列来处理，
而不是等待所有的工作处理完成才返回。将每一个任务封装成一个消息，将它发送到一个队列。
然后后台运行一个工作进程，通过将队列中的消息pop出来进行处理，可以同时有多个工作进程。

这种情况在web应用程序里面经常出现，因为很多时候无法在一个短时间的HTTP连接请求内完成一个耗时的任务。

这里模拟一个长时间任务，用发送的消息中含有的点数`.`来表示耗时几秒，一个点表示一秒钟。

```python new_task.py
import pika

connection = pika.BlockingConnection(pika.ConnectionParameters(host='192.168.217.161', port=5673))
channel = connection.channel()

channel.queue_declare(queue='task_queue')

messages = ['python new_task.py First message.',
            'python new_task.py Second message..',
            'python new_task.py Third message...',
            'python new_task.py Fourth message....',
            'python new_task.py Fifth message.....',
            'python new_task.py Sixth message......']
for m in messages:
    channel.basic_publish(exchange='',
                          routing_key='task_queue',
                          body=m,
                          properties=pika.BasicProperties(
                              delivery_mode=2,  # make message persistent
                          ))
    print(" [x] Sent %r" % m)

connection.close()
```

工作进程如下：

```python worker.py
import pika
import time

connection = pika.BlockingConnection(pika.ConnectionParameters(host='192.168.217.161', port=5673))
channel = connection.channel()

channel.queue_declare(queue='task_queue')

def callback(ch, method, properties, body):
    print(" [x] Received %r" % body)
    time.sleep(body.count(b'.'))
    print(" [x] Done")

channel.basic_consume(callback,
                      queue='task_queue',
                      no_ack=True)

print(' [*] Waiting for messages. To exit press CTRL+C')
channel.start_consuming()
```

运行的时候，可以同时开两个窗口运行worker进程，然后再运行`new_task.py`，最后效果：

消息发送：

```
 [x] Sent 'python new_task.py First message.'
 [x] Sent 'python new_task.py Second message..'
 [x] Sent 'python new_task.py Third message...'
 [x] Sent 'python new_task.py Fourth message....'
 [x] Sent 'python new_task.py Fifth message.....'
 [x] Sent 'python new_task.py Sixth message......'
```

工作进程1：

```
 [*] Waiting for messages. To exit press CTRL+C
 [x] Received b'python new_task.py First message.'
 [x] Done
 [x] Received b'python new_task.py Third message...'
 [x] Done
 [x] Received b'python new_task.py Fifth message.....'
 [x] Done
```

工作进程2：

```
 [*] Waiting for messages. To exit press CTRL+C
 [x] Received b'python new_task.py Second message..'
 [x] Done
 [x] Received b'python new_task.py Fourth message....'
 [x] Done
 [x] Received b'python new_task.py Sixth message......'
 [x] Done
```

### 消息确认

RabbitMQ利用`Message acknowledgment`来确保消息已经被正常并且处理完成，防止消息丢失。

默认情况下是开启的，前面的例子，我关闭了它，使用`no_ack=True`，现在我来打开它，
当处理完成后，发送一个正确的消息确认：

```python
def callback(ch, method, properties, body):
    print " [x] Received %r" % (body,)
    time.sleep( body.count('.') )
    print " [x] Done"
    ch.basic_ack(delivery_tag = method.delivery_tag)

channel.basic_consume(callback,
                      queue='task_queue')
```

这个时候，当你运行某个worker，中途kill掉这个进程，
你会发现worker里面没有处理完成的消息会重新跑到另外的worker上面运行。

### 消息持久化

上面的消息确认可以让我们在worker挂掉时候也能确保消息的安全性，但是如果rabbitmq服务器挂掉了呢，
那消息都没了。服务器上面的消息通过持久化来保证安全性。我们需要同时什么队列和消息是持久化的。

第一步申明队列是持久化的：

```python
channel.queue_declare(queue='task_queue', durable=True)
```

第二步申明消息也是持久化的：

```python
channel.basic_publish(exchange='',
      routing_key='task_queue',
      body=m,
      properties=pika.BasicProperties(
          delivery_mode=2,  # make message persistent
      ))
```

不过这个并不会保证严格的持久化，对于一般情况已经可以满足，如给你需要完全的持久化，
请参考[publisher confirms](https://www.rabbitmq.com/confirms.html)

### 消息分发策略

默认策略是只要消息来了我就分给某个worker，而不会去管那个worker是否已经完成了任务。
可通过`basic_qos`方法，并指定只有收到了消息处理应答后才将消息发送给worker。

```python
channel.basic_qos(prefetch_count=1)
```
