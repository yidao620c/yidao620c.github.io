---
title: JDK8中字符串常量池详解
date: 2021-04-01 19:33:21 +0800
toc: true
categories: [ java ]
tags: [ jvm ]
---

在JDK1.8 Hotspot移除了永久代用元空间(Metaspace)取而代之, 这时候字符串常量池还在堆, 运行时常量池还在方法区, 
只不过方法区的实现从永久代变成了元空间(Metaspace) ，元空间使用的是直接内存，跟JVM内存无关。

字符串常量池中同时存在字符串常量和字符串引用。直接赋值和`new String("xxx")`构造函数都可能导致常量池中生成字符串常量;
而intern()方法会尝试将堆中对象的引用放入常量池。

字符串的创建和使用几种典型常见。

<!-- more -->

## 1、只在常量池上创建常量

![img.png](https://xnstatic-1253397658.file.myqcloud.com/jdk8-string01.png)

## 2、只在堆上创建对象

![img.png](https://xnstatic-1253397658.file.myqcloud.com/jdk8-string02.png)

## 3、在堆上创建对象，在常量池上创建常量

![img.png](https://xnstatic-1253397658.file.myqcloud.com/jdk8-string03.png)

## 4、在堆上创建对象，在常量池上创建引用

![img.png](https://xnstatic-1253397658.file.myqcloud.com/jdk8-string04.png)

解释如下：

* 常量池有两种情况：`引用` 或 `常量`。如果该位置已经是引用或常量了，之后的操作都不会改变里面的情况！！！
* 调用`intern()`: 如果常量池里是空的，就创建引用（指向堆，参考结论4）；非空，不操作。返回值都是常量池里的内容。
* 堆中可以有任意个相同的字符串，常量池只能有一个（引用 或 常量）。
* `""`和`intern()`其实很像。区别就是在常量池为空时，`""`是把值加进去，`intern()`是把引用加进去。

## 举例说明
例子1：
``` java
String s1 = new String("zxy");    // 场景3
s1.intern(); // 常量池非空，返回值是常量池里的内容
String s2 = "zxy"; // 常量池非空，返回值是常量池里的内容
System.out.println(s1 == s2); //false
System.out.println(s1.intern() == s2); // true
```
![img.png](https://xnstatic-1253397658.file.myqcloud.com/jdk8-string05.png)

例子2：
``` java
String s1 = "zxy"; // 加到常量池
String s2 = new String("zxy"); // 加到堆，常量池有东西所以啥也不干
System.out.println(s1 == s2); // false
System.out.println(s1 == s2.intern()); // true 常量池非空，intern返回常量池里的内容
```
![img.png](https://xnstatic-1253397658.file.myqcloud.com/jdk8-string06.png)

