---
title: Java内存区域与内存溢出异常
date: 2021-04-03 00:42:11 +0800
toc: true
categories: [ java ]
tags: [ jvm ]
---

Java与C++之间有一堵由内存动态分配和垃圾收集技术所围成的高墙，墙外面的人想进去，墙里面的人却想出来。
按照《Java虚拟机规范(第2版)》的规定，Java虚拟机所管理的内存将包括以下几个运行时数据区域。

来个图更加直观点，如下图所示：
![](https://xnstatic-1253397658.file.myqcloud.com/jvm.png)
<!-- more -->

### 程序计数器

Program Counter Register是一块较小的内存空间，它的作用可以看做是当前线程所执行的字节码的行号指示器。
每个线程都有一个独立的程序计数器，各个线程之间计数器互不影响，独立存储。
此内存区域是唯一一个在Java虚拟机规范中没有规定任何OutOfMemoryError情况的区域。

### Java虚拟机栈

也是线程私有的，它的生命周期与线程相同。每个方法被执行的时候会同时创建一个栈帧（Stack
Frame）用于存储局部变量表、操作栈、动态链接、方法出口等信息。每个方法被调用直至执行完成的过程，就对应着一个栈帧在虚拟机中从入栈到出栈的过程。
如果线程请求栈深度大于虚拟机所允许的深度，抛出StackOverflowError
如果虚拟机栈可以动态扩展，扩展时无法申请到足够的内存时会抛出OutOfMemoryError

### 本地方法栈

Native Method Stacks与虚拟机栈所发挥的作用是非常相似的，只不过一个是执行Java方法，一个是Nataive方法，HotSpot虚拟机直接将两者合二为一了

### Java堆

Java堆是被所有线程共享的一块内存区域，在虚拟机启动时创建。Java堆是垃圾收集器管理的主要区域，很多时候称为GC堆。
如果在堆中没有内存完成实例分配，并且堆也无法再扩展时，将会抛出OutOfMemoryError

### 方法区

Method Area与Java堆一样，是各个线程共享的内存区域，它用于存储已被虚拟机加载的类信息、常量、静态变量、JIT编译后的代码等数据。
当方法区无法满足内存分配需求时，将抛出OutOfMemoryError。

### 运行时常量池

Runtime Constant Pool是方法区的一部分。用于存放编译器生成的各种字面量和符号引用，这部分内容将在类加载后存放到方法区的运行时常量池中。
当常量池无法再申请到内存时会抛出OutOfMemoryError异常。

### 直接内存

Direct
Memory并不是虚拟机运行时数据区的一部分，也不是Java虚拟机规范中定义的内存区域，但是这部分也是频繁使用。在Java的NIO中使用到，服务器管理员忽略直接内存后果是，各个内存区域总和大于物理内存限制，从而导致动态扩展时出现OutOfMemoryError异常。

### OutOfMemoryError异常

#### Java堆溢出

Java堆用于存储对象实例，我们只要不断创建对象，并且保证GC Roots到对象之间有可达路径来避免GC清除这些对象，就会在对象数量到达最大堆的容量限制后产生内存溢出异常。

```
VM Args: -Xms10m -Xmx10m -XX:+HeapDumpOnOutOfMemoryError
```

`XX:+HeapDumpOnOutOfMemoryError`这个参数可以让虚拟机在出现内存溢出异常时Dump出当前的内存堆转储快照以便事后进行分析。

```java
import java.util.ArrayList;
import java.util.List;

/**
 * VM Args: -Xms10m -Xmx10m -XX:+HeapDumpOnOutOfMemoryError
 * @author Administrator
 *
 */
public class HeapOOM {
	static class OOMObject{
		private String name;
		public OOMObject(String name) {
			this.name = name;
		}
	}
	public static void main(String[] args) {
		List<OOMObject> list = new ArrayList<OOMObject>();
		long i = 1;
		while(true) {
			list.add(new OOMObject("IpConfig..." + i++));
		}
	}
}
```

抛出的异常：

```
Dumping heap to java_pid27828.hprof ...
Heap dump file created [14123367 bytes in 0.187 secs]
Exception in thread "main" java.lang.OutOfMemoryError: Java heap space
at java.lang.AbstractStringBuilder.<init>(AbstractStringBuilder.java:45)
at java.lang.StringBuilder.<init>(StringBuilder.java:92)
at com.baoxian.HeapOOM.main(HeapOOM.java:22)
```

注：出现Java堆内存溢出时，异常堆栈信息 java.lang.OutOfMemoryError 后面会紧跟着 Java heap space。

要解决这个异常，一般手段是首先通过内存映像分析工具比如Eclipse Memory
Analyzer对dump出来的堆转储快照进行分析，重点是确认内存中对象是否是必要的，也就是要弄清楚到底是出现了内存泄露 Memory
Leak还是内存溢出 Memory Overflow。
如果是内存泄露，可进一步通过工具查看泄露对象到GC Roots的引用链。于是就能找到泄露对象时通过怎样的路径与GC
Roots相关联并导致垃圾收集器无法自动回收它们。掌握了泄露对象的类型信息，以及GC Roots引用链的信息，就可以比较准确的定位出泄露代码的位置了。
如果不存在泄露，那么就该修改-Xms 和 -Xms堆参数看能否加大点。

#### 虚拟机栈和本地方法栈溢出

-Xoss参数设置本地方法栈大小，对于HotSpot没用。栈容量只由-Xss参数设定

```java
/**
 * VM Args: -Xss128k
 * @author Administrator
 *
 */
public class JavaVMStackSOF {
	private int stackLength = 1;
	public void stackLeak() {
		stackLength++;
		stackLeak();
	}
	public static void main(String[] args) throws Throwable{
		JavaVMStackSOF oom = new JavaVMStackSOF();
		try {
			oom.stackLeak();
		} catch (Throwable e) {
			System.out.println("stack length: " + oom.stackLength);
			throw e;
		}

	}

}
```

抛出异常：

```
stack length: 1007
Exception in thread "main" java.lang.StackOverflowError
at com.baoxian.JavaVMStackSOF.stackLeak(JavaVMStackSOF.java:11)
at com.baoxian.JavaVMStackSOF.stackLeak(JavaVMStackSOF.java:12)
at com.baoxian.JavaVMStackSOF.stackLeak(JavaVMStackSOF.java:12)
at com.baoxian.JavaVMStackSOF.stackLeak(JavaVMStackSOF.java:12)。。。。
```

#### 运行时常量池溢出

运行时常量池分配在方法区内，可以通过 -XX:PermSize和 -XX:MaxPermSize限制方法区大小，从而间接限制其中常量池的容量。

```java
import java.util.ArrayList;
import java.util.List;

/**
 * VM Args: -XX:PermSize=10M -XX:MaxPermSize=10M
 * @author Administrator
 *
 */
public class RuntimeConstantPoolOOM {
	public static void main(String[] args) {
		// 使用List保持着常量池引用，避免Full GC回收常量池行为
		List<String> list = new ArrayList<String>();
		// 10MB的PermSize在integer范围内足够产生OOM了
		int i = 0;
		while (true) {
			list.add(String.valueOf(i++).intern());
		}
	}
}
```

抛出异常：

```
Exception in thread "main" java.lang.OutOfMemoryError: PermGen space
at java.lang.String.intern(Native Method)
at com.baoxian.RuntimeConstantPoolOOM.main(RuntimeConstantPoolOOM.java:18)
```

运行时常量池溢出，在`java.lang.OutOfMemoryError`后面紧跟着是`PermGen space`

#### 方法区溢出

方法区用于存放Class的相关信息，如类名、访问修饰符、常量池、字段描述符、方法描述等。对于这个区域的测试，基本的思路是运行时产生大量的类去填满方法区，直到溢出。比如动态代理会生成动态类。
使用CGLib技术直接操作字节码运行，生成大量的动态类。当前很多主流框架如Spring和Hibernate对类进行增强都会使用CGLib这类字节码技术，增强的类越多，就需要越大的方法区来保证动态生成的Class可以加载入内存。

异常：

```
Exception in thread "main" java.lang.OutOfMemoryError: PermGen space
at java.lang.String.intern(Native Method)
```

同样，跟常量池一样，都是PermGen space字符串出现。
方法区溢出也是一种常见的内存溢出异常，一个类如果要被垃圾收集器回收，判定条件是非常苛刻的。在经常动态生成大量Class的应用中，需要特别注意类的回收状况。这类场景除了上面提到的程序使用GCLib字节码技术外，常见的还有：
大量JSP或动态产生的JSP文件的应用（JSP第一次运行时需要编译为Java类）、基于OSGi应用等。

#### 本机直接内存溢出

DirectMemory容量可以通过-XX:MaxDirectMemorySize指定，如果不指定，则默认与Java堆的最大值-Xmx指定一样。

```java
/**
 * VM Args: -Xmx20M -XX:MaxDirectMemorySize=10M
 * @author Administrator
 *
 */
public class DirectMemoryOOM {
	private static final int _1MB = 1024 * 1024;
	public static void main(String[] args) {
		Field unsafeField = Unsafe.class.getDeclaredFields()[0];
		unsafeField.setAccessible(true);
		Unsafe unsafe = (Unsafe) unsafeField.get(null);
		while(true) {
			unsafe.allocateMemory(_1MB);
		}
	}
}
```

在OutOfMemoryError后面不会有任何东西了，这就是DirectMemory内存溢出了。

