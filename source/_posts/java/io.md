---
layout: post
title: "聊聊java中的几种I/O模型"
date: 2018-04-02 12:07:42 +0800
toc: true
categories: java
tags: java
---

同步、异步、阻塞、非阻塞都是和I/O（输入输出）有关的概念，最简单的文件读取就是I/O操作。而在文件读取这件事儿上，可以有多种方式。

本篇会先介绍一下I/O的基本概念，通过一个生活例子来分别解释下这几种I/O模型，并聊一聊在Java中如何实现这几种模型，
最后还会详细讲解在Java中怎样实现Reactor模式。<!--more-->

## 基本概念

在解释I/O模型之前，我先说明一下几个操作系统的概念

### 进程切换

为了控制进程的执行，内核必须有能力挂起正在CPU上运行的进程，并恢复以前挂起的某个进程的执行，这种行为被称为进程切换。
因此可以说，任何进程都是在操作系统内核的支持下运行的，是与内核紧密相关的。

### 进程阻塞

正在执行的进程，由于期待的某些事件未发生，如请求系统资源失败、等待某种操作的完成、新数据尚未到达或无新工作做等，
则由系统自动执行阻塞原语(Block)，使自己由运行状态变为阻塞状态。可见，进程的阻塞是进程自身的一种主动行为，
也因此只有处于运行态的进程（获得CPU），才可能将其转为阻塞状态。当进程进入阻塞状态，是不占用CPU资源的。

### 文件描述符fd

文件描述符（File descriptor）是计算机科学中的一个术语，是一个用于表述指向文件的引用的抽象化概念。

文件描述符在形式上是一个非负整数。实际上，它是一个索引值，指向内核为每一个进程所维护的该进程打开文件的记录表。
当程序打开一个现有文件或者创建一个新文件时，内核向进程返回一个文件描述符。
在程序设计中，一些涉及底层的程序编写往往会围绕着文件描述符展开。文件描述符这一概念往往只适用于UNIX、Linux这样的操作系统。

### 缓存I/O

缓存 I/O 又被称作标准 I/O，大多数文件系统的默认 I/O 操作都是缓存 I/O。在 Linux 的缓存 I/O 机制中，
操作系统会将 I/O 的数据缓存在文件系统的页缓存（ page cache ）中，也就是说，数据会先被拷贝到操作系统内核的缓冲区中，
然后才会从操作系统内核的缓冲区拷贝到应用程序的地址空间。

缓存 I/O 的缺点：

数据在传输过程中需要在应用程序地址空间和内核进行多次数据拷贝操作，这些数据拷贝操作所带来的 CPU 以及内存开销是非常大的。

下面我以一个生活中烧开水的例子来形象解释一下同步、异步、阻塞、非阻塞概念。

### 同步和异步

说到烧水，我们都是通过热水壶来烧水的。在很久之前，科技还没有这么发达的时候，如果我们要烧水，
需要把水壶放到火炉上，我们通过观察水壶内的水的沸腾程度来判断水有没有烧开。

随着科技的发展，现在市面上的水壶都有了提醒功能，当我们把水壶插电之后，水壶水烧开之后会通过声音提醒我们水开了。

对于烧水这件事儿来说，传统水壶的烧水就是同步的，高科技水壶的烧水就是异步的。

**同步请求**

A调用B，B的处理是同步的，在处理完之前他不会通知A，只有处理完之后才会明确的通知A。

**异步请求**

A调用B，B的处理是异步的，B在接到请求后先告诉A我已经接到请求了，然后异步去处理，处理完之后通过回调等方式再通知A。

所以说，同步和异步最大的区别就是被调用方的执行方式和返回时机。
同步指的是被调用方做完事情之后再返回，异步指的是被调用方先返回，然后再做事情，做完之后再想办法通知调用方。

### 阻塞和非阻塞

还是那个烧水的例子，当你把水放到水壶里面，按下开关后，你可以坐在水壶前面，别的事情什么都不做，
一直等着水烧好。你还可以先去客厅看电视，等着水开就好了。

对于你来说，坐在水壶前面等就是阻塞的，去客厅看电视等着水开就是非阻塞的。

**阻塞请求**

A调用B，A一直等着B的返回，别的事情什么也不干。

**非阻塞请求**

A调用B，A不用一直等着B的返回，先去忙别的事情了。

所以说，阻塞和非阻塞最大的区别就是在被调用方返回结果之前的这段时间内，调用方是否一直等待。
阻塞指的是调用方一直等待别的事情什么都不做。非阻塞指的是调用方先去忙别的事情。

### 阻塞、非阻塞和同步、异步的区别

首先，前面已经提到过，阻塞、非阻塞和同步、异步其实针对的对象是不一样的。

给我大声念三遍下面的句子

*阻塞、非阻塞说的是调用者。同步、异步说的是被调用者。*

*阻塞、非阻塞说的是调用者。同步、异步说的是被调用者。*

*阻塞、非阻塞说的是调用者。同步、异步说的是被调用者。*

有人认为阻塞和同步是一回事儿，非阻塞和异步是一回事。但是这是不对的。

========> 同步包含阻塞和非阻塞 <===========

我们是用传统的水壶烧水。在水烧开之前我们一直做在水壶前面，等着水开。这就是阻塞的。

我们是用传统的水壶烧水。在水烧开之前我们先去客厅看电视了，但是水壶不会主动通知我们，
需要我们时不时的去厨房看一下水有没有烧开，这就是非阻塞的。

========> 异步包含阻塞和非阻塞 <===========

我们是用带有提醒功能的水壶烧水。在水烧发出提醒之前我们一直做在水壶前面，等着水开。这就是阻塞的。

我们是用带有提醒功能的水壶烧水。在水烧发出提醒之前我们先去客厅看电视了，等水壶发出声音提醒我们。这就是非阻塞的。

## Unix中的五种I/O模型

对于一次I/O访问（以read举例），数据会先被拷贝到操作系统内核的缓冲区中，然后才会从操作系统内核的缓冲区拷贝到应用程序的地址空间。
所以说，当一个read操作发生时，它会经历两个阶段：

* 第一阶段：等待数据准备 (Waiting for the data to be ready)。
* 第二阶段：将数据从内核拷贝到进程中 (Copying the data from the kernel to the process)。

对于socket流而言

* 第一步：通常涉及等待网络上的数据分组到达，然后被复制到内核的某个缓冲区。
* 第二步：把数据从内核缓冲区复制到应用进程缓冲区。

Unix下五种I/O模型：

1. 阻塞 I/O
1. 非阻塞 I/O
1. I/O 多路复用（select和poll）
1. 信号驱动 I/O（SIGIO）
1. 异步 I/O

### 阻塞I/O

阻塞I/O下请求无法立即完成则保持阻塞。阻塞I/O分为如下两个阶段。

* 阶段1：等待数据就绪。网络 I/O 的情况就是等待远端数据陆续抵达；磁盘I/O的情况就是等待磁盘数据从磁盘上读取到内核态内存中。
* 阶段2：数据拷贝。出于系统安全，用户态的程序没有权限直接读取内核态内存，因此内核负责把内核态内存中的数据拷贝一份到用户态内存中。

### I/O多路复用

我这里只想重点解释一下I/O多路复用这种模型，因为现在用的最多。

I/O多路复用是一种异步阻塞I/O，会用到select或者poll函数，或者进阶版的epoll，这些函数也会使线程阻塞，
但是和阻塞I/O所不同的是，这几个函数可以同时阻塞多个I/O操作。而且可以同时对多个读操作，多个写操作的I/O函数进行检测，
直到有数据可读或可写时，才真正调用I/O操作函数。

从流程上来看，使用select函数进行I/O请求和同步阻塞模型没有太大的区别，甚至还多了添加监视Channel，
以及调用select函数的额外操作，增加了额外工作。但是，使用select以后最大的优势是用户可以在一个线程内同时处理多个Channel的I/O请求。
用户可以注册多个Channel，然后不断地调用select读取被激活的Channel，即可达到在同一个线程内同时处理多个I/O请求的目的。
而在同步阻塞模型中，必须通过多线程的方式才能达到这个目的。

调用select/poll该方法由一个用户态线程负责轮询多个Channel，直到某个阶段1的数据就绪，
再通知实际的用户线程执行阶段2的拷贝。通过一个专职的用户态线程执行非阻塞I/O轮询，模拟实现了阶段一的异步化。

## Java中四种I/O模型

上一章所述Unix中的五种I/O模型，除信号驱动I/O外，Java对其它四种I/O模型都有所支持。

1. Java最早提供的blocking I/O即是阻塞I/O
1. NIO即是非阻塞I/O，
1. 通过NIO实现的Reactor模式即是I/O复用模型的实现
1. 通过AIO实现的Proactor模式即是异步I/O模型的实现

## Reactor模式

Java中通过Reactor模式实现I/O多路复用，本文最后部分讲解该模式的实现机制。

Reactor模式又分为三种实现方式

### 经典模式

精典的Reactor模式示意图如下所示。

![](https://xnstatic-1253397658.file.myqcloud.com/reactor01.png)

在Reactor模式中，包含如下角色

* Reactor 将I/O事件发派给对应的Handler
* Acceptor 处理客户端连接请求
* Handlers 执行非阻塞读/写

最简单的Reactor模式实现代码如下所示

``` java
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

``` java
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

``` java
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

``` java
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

``` java
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

