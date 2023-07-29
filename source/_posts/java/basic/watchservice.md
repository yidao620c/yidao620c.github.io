---
title: 使用WatchService监听文件变化
date: 2017-09-21 16:12:22 +0800
toc: true
categories: [ java ]
tags: [ java ]
---

在Java 7发布的新的IO框架中，除了大家都熟知的 `FileVisitor` 接口外，还有个 `WatchService` 接口经常被人忽视掉。
这个类可以让你实时的监控操作系统中文件的变化，包括创建、更新和删除事件。

`WatchService` 用来观察被注册了的对象的变化和事件。它和`Watchable`两个接口的配合使用，
WatchService类似于在观察者模式中的观察者，Watchable类似域观察者模式中的被观察者。

而Java 7中的`java.nio.file.Path`类就实现了Watchable接口，这样的话，和Path类一起可实现观察者模式。
<!-- more -->

## 基本使用

要使用WatchService，首先必须创建一个实例，使用 ` java.nio.file.FileSystems` 类：

```java
WatchService watchService = FileSystems.getDefault().newWatchService();
```

接下来，我们初始化一个被监控文件夹的Path类：

```java
Path path = Paths.get("pathToDir");
```

然后，将这个Path注册到 `WatchService` 中去：

```java
WatchKey watchKey = path.register(
  watchService, StandardWatchEventKinds...);
```

注意，注册方法第二个参数是一个可变长参数，也就是可以注册多个事件类型。

`StandardWatchEventKinds` 有如下四种类型：

1. ENTRY_CREATE 创建事件，可是新建一个文件或重命名文件
1. ENTRY_MODIFY 修改事件，文件内容被修改，有些系统上面文件属性被修改也会触发
1. ENTRY_DELETE 删除事件，文件被删除或被重命名
1. OVERFLOW 如果丢失或放弃事件时被触发，我们一般不会关注这个类型

## WatchKey

`WatchKey` 类代表了这个监听服务的注册，可以用它来获取事件的各个属性。一般来讲我们会通过如下方法获取这个类实例：

```java
WatchKey watchKey = watchService.take();
```

这个方法会一直阻塞，知道某个事件到来。

有一点非常重要的需要记住，一旦 `WatchKey` 实例通过poll或take返回后，它再也不会捕获任何事件，触发你调用reset方法：

```java
watchKey.reset();
```

这样的话就能将这个 `WatchKey` 又放回到watch service队列中去，可以重新等待新的事件了。

所以一般来讲这个服务代码形式会写成如下形式：

```java
WatchKey key;
while ((key = watchService.take()) != null) {
    for (WatchEvent<?> event : key.pollEvents()) {
        //process
    }
    key.reset();
}
```

解释一下运行过程，take()方法会一直阻塞直到返回一个WatchKey实例，然后进入循环内部，
pollEvents()方法会返回当前Key上面所有事件列表，我们一个个去处理这些事件。

所有事件处理完成后，必须调用reset方法，让这个key重新进入监听队列。

## 一个完整例子

下面通过一个完整例子演示WatchService的用法，我们监听用户家目录的文件变动，打印相应的信息。

```java
public class DirectoryWatcherExample {
 
    public static void main(String[] args) {
        WatchService watchService
          = FileSystems.getDefault().newWatchService();
 
        Path path = Paths.get(System.getProperty("user.home"));
 
        path.register(
          watchService, 
            StandardWatchEventKinds.ENTRY_CREATE, 
              StandardWatchEventKinds.ENTRY_DELETE, 
                StandardWatchEventKinds.ENTRY_MODIFY);
 
        WatchKey key;
        while ((key = watchService.take()) != null) {
            for (WatchEvent<?> event : key.pollEvents()) {
                System.out.println(
                  "Event kind:" + event.kind() 
                    + ". File affected: " + event.context() + ".");
            }
            key.reset();
        }
    }
}
```

测试一下，在用户目录新建一个文件，然后重命名为`www.txt`，看看输出结果：

```
Event kind:ENTRY_CREATE. File affected: 新建文本文档.txt.
Event kind:ENTRY_MODIFY. File affected: 新建文本文档.txt.
Event kind:ENTRY_DELETE. File affected: 新建文本文档.txt.
Event kind:ENTRY_CREATE. File affected: www.txt.
```

上面两行是新建文件输出，可以看到产生两个事件。下面是重命名产生两个事件，一个是原文件的删除事件，一个是新文件的新建事件。
