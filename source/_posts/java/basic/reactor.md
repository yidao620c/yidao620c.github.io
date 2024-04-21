---
title: 聊聊Reactor I/O模型
date: 2018-04-05 09:23:12 +0800
toc: true
categories: [ java ]
tags: [ java ]
---

上一篇介绍了Unix系统支持的I/O模型，以及相应的在Java中的实现方式。本篇重点讲解一下Reactor模型原理和实现机制。

I/O多路复用又被称为"事件驱动"，就是通过一种机制，一个进程可以监视多个描述符，一旦某个描述符就绪（一般是读就绪或者写就绪），
能够通知程序进行相应的读写操作，技术上是通过调用操作系统的select、pselect、poll、epoll来实现。

与多进程和多线程技术相比，I/O多路复用技术的最大优势是系统开销小，系统不必创建进程/线程，也不必维护这些进程/线程，从而大大减小了系统的开销。

Reactor是一种应用在服务器端的开发模式，目的是提高服务端程序的并发能力，其实就是实现了I/O多路复用这种I/O模型。
<!-- more -->

## 图解Reactor模式

为了形象的解释这个模式，我用一个餐厅点餐的例子来直观的说明一下。

### 传统的多线程方式

每个顾客都有自己的服务团队（线程），在人少的情况下是可以良好的运作的，流程如下：

1. 服务员给出菜单，并等待点菜
1. 顾客查看菜单，并点菜
1. 服务员把菜单交给厨师，厨师照着做菜
1. 厨师做好菜后端到餐桌上

如下图所示

![](https://xnstatic-1253397658.file.myqcloud.com/reactor04.png)

现在餐厅的口碑好，顾客人数不断增加，这时服务员就有点处理不过来了。这时老板发现，每个服务员在服务完客人后，都要去休息一下，
因此老板就说，"你们都别休息了，在旁边待命"。这样可能 10 个服务员也来得及服务 20 个顾客了。这也是"线程池"的方式，
通过重用线程来减少线程的创建和销毁时间，从而提高性能。

### Reactor方式

但是客人又进一步增加了，仅仅靠剥削服务员的休息时间也没有办法服务这么多客人。老板仔细观察，
发现其实服务员并不是一直在干活的，大部分时间他们只是站在餐桌旁边等客人点菜。

于是老板就对服务员说，客人点菜的时候你们就别傻站着了，先去服务其它客人，有客人点好的时候喊你们再过去。对应于下图：

![](https://xnstatic-1253397658.file.myqcloud.com/reactor05.png)

最后，老板发现根本不需要那么多的服务员，于是裁了一波员，最终甚至可以只有一个服务员。

这就是 Reactor 模式的核心思想：减少等待。当遇到需要等待 IO 时，先释放资源，而在 IO 完成时，
再通过事件驱动 (event driven) 的方式，继续接下来的处理。从整体上减少了资源的消耗。

## Java中实现Reactor模式

本篇详细讲解一下Java中如何来实现Reactor模式。

Reactor模式又分为三种实现方式

### 经典模式

精典的Reactor模式示意图如下所示。

![](https://xnstatic-1253397658.file.myqcloud.com/reactor01.png)

在Reactor模式中，包含如下角色

* Reactor 将I/O事件发派给对应的Handler
* Acceptor 处理客户端连接请求
* Handlers 执行非阻塞读/写

最简单的Reactor模式实现代码如下所示

```java
public class NIOServer {
  private static final Logger LOGGER = LoggerFactory.getLogger(NIOServer.class);
  public static void main(String[] args) throws IOException {
    Selector selector = Selector.open();
    ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();
    serverSocketChannel.configureBlocking(false);
    serverSocketChannel.bind(new InetSocketAddress(1234));
    serverSocketChannel.register(selector, SelectionKey.OP_ACCEPT);
    while (selector.select() > 0) {
      Set<SelectionKey> keys = selector.selectedKeys();
      Iterator<SelectionKey> iterator = keys.iterator();
      while (iterator.hasNext()) {
        SelectionKey key = iterator.next();
        iterator.remove();
        if (key.isAcceptable()) {
          ServerSocketChannel acceptServerSocketChannel = (ServerSocketChannel) key.channel();
          SocketChannel socketChannel = acceptServerSocketChannel.accept();
          socketChannel.configureBlocking(false);
          LOGGER.info("Accept request from {}", socketChannel.getRemoteAddress());
          socketChannel.register(selector, SelectionKey.OP_READ);
        } else if (key.isReadable()) {
          SocketChannel socketChannel = (SocketChannel) key.channel();
          ByteBuffer buffer = ByteBuffer.allocate(1024);
          int count = socketChannel.read(buffer);
          if (count <= 0) {
            socketChannel.close();
            key.cancel();
            LOGGER.info("Received invalide data, close the connection");
            continue;
          }
          LOGGER.info("Received message {}", new String(buffer.array()));
        }
        keys.remove(key);
      }
    }
  }
}
```

从上示代码中可以看到，多个Channel可以注册到同一个Selector对象上，实现了一个线程同时监控多个请求状态（Channel）。
同时注册时需要指定它所关注的事件，例如上示代码中socketServerChannel对象只注册了OP_ACCEPT事件，
而socketChannel对象只注册了OP_READ事件。

selector.select()是阻塞的，当有至少一个通道可用时该方法返回可用通道个数。同时该方法只捕获Channel注册时指定的所关注的事件。

### 多工作线程模式

经典Reactor模式中，尽管一个线程可同时监控多个请求（Channel），但是所有读/写请求以及对新连接请求的处理都在同一个线程中处理，
无法充分利用多CPU的优势，同时读/写操作也会阻塞对新连接请求的处理。因此可以引入多线程，并行处理多个读/写操作，如下图所示。

![](https://xnstatic-1253397658.file.myqcloud.com/reactor02.png)

多线程Reactor模式示例代码如下所示：

```java
public class NIOServer {
  private static final Logger LOGGER = LoggerFactory.getLogger(NIOServer.class);
  public static void main(String[] args) throws IOException {
    Selector selector = Selector.open();
    ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();
    serverSocketChannel.configureBlocking(false);
    serverSocketChannel.bind(new InetSocketAddress(1234));
    serverSocketChannel.register(selector, SelectionKey.OP_ACCEPT);
    while (true) {
      if(selector.selectNow() < 0) {
        continue;
      }
      Set<SelectionKey> keys = selector.selectedKeys();
      Iterator<SelectionKey> iterator = keys.iterator();
      while(iterator.hasNext()) {
        SelectionKey key = iterator.next();
        iterator.remove();
        if (key.isAcceptable()) {
          ServerSocketChannel acceptServerSocketChannel = (ServerSocketChannel) key.channel();
          SocketChannel socketChannel = acceptServerSocketChannel.accept();
          socketChannel.configureBlocking(false);
          LOGGER.info("Accept request from {}", socketChannel.getRemoteAddress());
          SelectionKey readKey = socketChannel.register(selector, SelectionKey.OP_READ);
          readKey.attach(new Processor());
        } else if (key.isReadable()) {
          Processor processor = (Processor) key.attachment();
          processor.process(key);
        }
      }
    }
  }
}
```

从上示代码中可以看到，注册完SocketChannel的OP_READ事件后，可以对相应的SelectionKey
绑定一个对象（本例中绑定了一个Processor对象，该对象处理读请求），并且在获取到可读事件后，可以取出该对象。

具体的读请求处理在如下所示的Processor类中。该类中设置了一个静态的线程池处理所有请求。而process方法并不直接处理I/O请求，
而是把该I/O操作提交给上述线程池去处理，这样就充分利用了多线程的优势，同时将对新连接的处理和读/写操作的处理放在了不同的线程中，
读/写操作不再阻塞对新连接请求的处理。

```java
public class Processor {
  private static final Logger LOGGER = LoggerFactory.getLogger(Processor.class);
  private static final ExecutorService service = Executors.newFixedThreadPool(16);
  public void process(SelectionKey selectionKey) {
    service.submit(() -> {
      ByteBuffer buffer = ByteBuffer.allocate(1024);
      SocketChannel socketChannel = (SocketChannel) selectionKey.channel();
      int count = socketChannel.read(buffer);
      if (count < 0) {
        socketChannel.close();
        selectionKey.cancel();
        LOGGER.info("{}\t Read ended", socketChannel);
        return null;
      } else if(count == 0) {
        return null;
      }
      LOGGER.info("{}\t Read message {}", socketChannel, new String(buffer.array()));
      return null;
    });
  }
}
```

### 多Reactor模式

Netty中使用的Reactor模式，引入了多Reactor，也即一个主Reactor负责监控所有的连接请求，多个子Reactor负责监控并处理读/写请求，
减轻了主Reactor的压力，降低了主Reactor压力太大而造成的延迟。并且每个子Reactor分别属于一个独立的线程，
每个成功连接后的Channel的所有操作由同一个线程处理。这样保证了同一请求的所有状态和上下文在同一个线程中，
避免了不必要的上下文切换，同时也方便了监控请求响应状态。

多Reactor模式示意图如下所示。

![](https://xnstatic-1253397658.file.myqcloud.com/reactor03.png)

多Reactor示例代码如下所示：

```java
public class NIOServer {
  private static final Logger LOGGER = LoggerFactory.getLogger(NIOServer.class);
  public static void main(String[] args) throws IOException {
    Selector selector = Selector.open();
    ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();
    serverSocketChannel.configureBlocking(false);
    serverSocketChannel.bind(new InetSocketAddress(1234));
    serverSocketChannel.register(selector, SelectionKey.OP_ACCEPT);
    int coreNum = Runtime.getRuntime().availableProcessors();
    Processor[] processors = new Processor[coreNum];
    for (int i = 0; i < processors.length; i++) {
      processors[i] = new Processor();
    }
    int index = 0;
    while (selector.select() > 0) {
      Set<SelectionKey> keys = selector.selectedKeys();
      for (SelectionKey key : keys) {
        keys.remove(key);
        if (key.isAcceptable()) {
          ServerSocketChannel acceptServerSocketChannel = (ServerSocketChannel) key.channel();
          SocketChannel socketChannel = acceptServerSocketChannel.accept();
          socketChannel.configureBlocking(false);
          LOGGER.info("Accept request from {}", socketChannel.getRemoteAddress());
          Processor processor = processors[(int) ((index++) % coreNum)];
          processor.addChannel(socketChannel);
          processor.wakeup();
        }
      }
    }
  }
}
```

如上代码所示，本文设置的子Reactor个数是当前机器可用核数的两倍（与Netty默认的子Reactor个数一致）。
对于每个成功连接的SocketChannel，通过round robin的方式交给不同的子Reactor。

子Reactor对SocketChannel的处理如下所示：

```java
public class Processor {
  private static final Logger LOGGER = LoggerFactory.getLogger(Processor.class);
  private static final ExecutorService service =
      Executors.newFixedThreadPool(2 * Runtime.getRuntime().availableProcessors());
  private Selector selector;
  public Processor() throws IOException {
    this.selector = SelectorProvider.provider().openSelector();
    start();
  }
  public void addChannel(SocketChannel socketChannel) throws ClosedChannelException {
    socketChannel.register(this.selector, SelectionKey.OP_READ);
  }
  public void wakeup() {
    this.selector.wakeup();
  }
  public void start() {
    service.submit(() -> {
      while (true) {
        if (selector.select(500) <= 0) {
          continue;
        }
        Set<SelectionKey> keys = selector.selectedKeys();
        Iterator<SelectionKey> iterator = keys.iterator();
        while (iterator.hasNext()) {
          SelectionKey key = iterator.next();
          iterator.remove();
          if (key.isReadable()) {
            ByteBuffer buffer = ByteBuffer.allocate(1024);
            SocketChannel socketChannel = (SocketChannel) key.channel();
            int count = socketChannel.read(buffer);
            if (count < 0) {
              socketChannel.close();
              key.cancel();
              LOGGER.info("{}\t Read ended", socketChannel);
              continue;
            } else if (count == 0) {
              LOGGER.info("{}\t Message size is 0", socketChannel);
              continue;
            } else {
              LOGGER.info("{}\t Read message {}", socketChannel, new String(buffer.array()));
            }
          }
        }
      }
    });
  }
}
```

在Processor中，同样创建了一个静态的线程池，且线程池的大小为机器核数的两倍。每个Processor实例均包含一个Selector实例。
同时每次获取Processor实例时均提交一个任务到该线程池，并且该任务正常情况下一直循环处理，不会停止。
而提交给该Processor的SocketChannel通过在其Selector注册事件，加入到相应的任务中。
由此实现了每个子Reactor包含一个Selector对象，并由一个独立的线程处理。

