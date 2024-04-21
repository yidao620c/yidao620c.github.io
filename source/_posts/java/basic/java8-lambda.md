---
title: Java8中的lambda表达式
date: 2015-05-08 15:42:11 +0800
toc: true
categories: [ java ]
tags: [ java ]
---

2014年3月18日，Oracle终于发布Java8正式版。在新的版本里面加入了很多特性，
总共增加了55个新特性，其中最最吸引人的就是Lambdas表达式和Stream函数式编程，本文详细讲解这两个特性。

其他特性比如日期API，泛型，反射，注解，集合框架，并发，Nashorn引擎等等这里暂时就不详细介绍了。
具体可以参考：<http://openjdk.java.net/projects/jdk8/features>

昨天参加了Oracle的Java8宣讲活动，有幸目睹了Simon Ritter的风采，写个总结来分享下。
<!-- more -->

### Java并发编程演变：

 版本  | 发布年份 | 并发技术                         
-----|------|------------------------------
 1.4 | 2002 | java.lang.Thread             
 5   | 2004 | java.util.concurrent(jsr166) 
 6   | 2006 | Phasers, etc(jsr166)         
 7   | 2011 | Fork/Join Framework(jsr166y) 
 8   | 2014 | Project Lambda               

先来一个小例子见识下Java8的威力！

### 传统的外部迭代处理代码

```java
List<Student> students = ...
double highestScore = 0.0;
for (Student s : students) {
    if (s.gradYear == 2011) {
        if (s.score > highestScore) {
            highestScore = s.score;
        }
    }
}
```

传统的外部迭代主要问题：

* 程序员自己控制迭代，容易出问题！
* 顺序执行：迭代从开始到结束一个一个的顺序迭代元素
* 线程不安全，由于业务逻辑依靠可修改变量，容易产生竞态问题

### 基于Inner Classes的内部迭代

```java
List<Student> students = ...
double highestScore = students.
        filter(new Predicate<Student>() {
            public boolean op(Student s) {
                return s.getGradYear() == 2011;
            }
        }).
        map(new Mapper<Student,Double>() {
            public Double extract(Student s) {
                return s.getScore();
            }
        }).
        max();
```

这种迭代形式已经具备了函数式特征。

优点：

* 迭代，过滤和累加器由核心库完成
* 遍历操作可以并行执行
* 遍历可以延迟执行
* 线程安全 – 因为客户端的逻辑是无状态的

缺点：

代码写的有点难看

### 基于Lambdas的内部迭代

```java
SomeList<Student> students = ...
double highestScore = students.
        filter(Student s -> s.getGradYear() == 2011).
        map(Student s -> s.getScore()).
        max();
```

这种写法可以算是完美了：^_^

* 可读性很好
* 更加抽象化
* 简单化后，自然就不容易出现bug了
* 不再依赖可变变量
* 很容易实现并行化

进入正题 ~~

### Lambda篇

Lambda表达式简单来讲就是匿名函数

* 就像一个方法一样，它又参数列表，一个返回类型，抛出的异常集和一个执行体
* 但是跟方法不同的是，它不跟任何Class关联。

也就是说，现在我们在Java的方法调用中不仅仅可以传值，还可以传动作(也就是函数)，这个有点类似于C语言的函数指针的概念了。

Lambda表达式的类型：

在Java中，到处都可以看到只有一个方法的接口，这种接口现在定义为函数式接口，
而Lambda表达式类型就是函数式接口，也就是只有一个方法的接口。

几个函数式接口的例子：

```java
interface Comparator<T> { boolean compare(T x, T y); }
interface FileFilter { boolean accept(File x); }
interface Runnable { void run(); }
interface ActionListener { void actionPerformed(…); }
interface Callable<T> { T call(); }
```

#### 局部变量捕获

Lambda表达式可以引用上下文中的final等效局部变量。

final等效指的是变量的用法是final的，而不必声明为final，比如变量只赋值一次，那么它就是final等效的。

```java
void expire(File root, long before) {
    root.listFiles(File p -> p.lastModified() <= before);
}
```

#### this关键字

Lambda表达式中的this指的是包含这个Lambda的外部对象，而不是Lambda本身。
永远记住，Lambda表达式类型其实就是一个函数式接口。

```java
class SessionManager {
    long before = ...;
    void expire(File root) {
        // refers to 'this.before', just like outside the lambda
        root.listFiles(File p -> checkExpiry(p.lastModified(), this.before));
    }
    boolean checkExpiry(long time, long expiry) { ... }
}
```

#### 类型推断

很多情况下，编译器都可以根据目标函数式接口的方法签名来推断参数类型。
在Collections接口中有个sort接口：

```java
static T void sort(List<T> l, Comparator<? super T> c);
```

正常来讲，应该这么写：

```java
List<String> list = getList();
Collections.sort(list, (String x, String y) -> x.length() - y.length());
```

借助类型推断，可以简化为：

```java
List<String> list = getList();
Collections.sort(list, (x, y) -> x.length() - y.length());
```

#### 方法引用

方法引用可以让我们将一个方法作为一个Lambda表达式重复利用。

比如，java.io.FileFilter作为一个函数式接口，仅有一个方法：

```java
boolean accept(File pathname);
```

正常的Lambda表达式用法：

```java
FileFilter x = File f -> f.canRead();
```

通过方法引用，可以简化为：

```java
FileFilter x = File::canRead;
```

方法引用语法格式有以下三种：

```
objectName::instanceMethod
ClassName::staticMethod
ClassName::instanceMethod
```

前两种方式类似，等同于把lambda表达式的参数直接当成instanceMethod\|staticMethod的参数来调用。

比如 `System.out::println` 等同于 `x->System.out.println(x);`
`Math::max`等同于`(x, y)->Math.max(x,y)`。

最后一种方式，等同于把lambda表达式的第一个参数当成instanceMethod的目标对象，
其他剩余参数当成该方法的参数。比如`String::toLowerCase`等同于`x->x.toLowerCase()`。

#### 构造器引用

构造器引用语法如下：`ClassName::new`，把lambda表达式的参数当成ClassName构造器的参数 。
例如`BigDecimal::new`等同于`x->new BigDecimal(x)`。

和方法引用类似，构造器引用示例：

正常的Lambda表达式的构造器示例：

```java
Factory<List<String>> f = () -> return new ArrayList<String>();
```

通过构造器引用，可以简化为：

```java
Factory<List<String>> f = ArrayList<String>::new;
```

#### 接口扩展

在Java中，接口是不能随便新增方法的，因为接口中一旦增加方法，那么所以实现类都必须重写。
可以在Interface中使用default关键字来增加一个新的接口方法，并提供一个默认实现。
接口的实现类可以不用管，也可以覆盖这个方法。

```java
interface Collection<E> {
    default Stream<E> stream() {
        return StreamSupport.stream(spliterator());
    }
}
```

还可以使用@FunctionalInterface这个注解来注解函数式接口，如果接口中多于一个抽象方法，编译器肯定报错。

更甚至，在Java8中，在接口中也可以增加静态方法了。

### Stream篇

#### 定义

许多的业务逻辑都需要聚集操作，比如按地区分类获取最优价值产品，按币种分类获取交易量。
之前版本的Java都是通过外部循环来完成这些操作，前面也说过了这种做法的很多弊端。

Java8给出完美解决方案：Lambda表达式+Stream API

Java中对Stream的定义：

```
A sequence of elements supporting sequential and parallel aggregate operations.
```

我们来解读一下上面的那句话：

```
- Stream是元素的集合，这点让Stream看起来用些类似Iterator；
– 可以支持顺序和并行的对原Stream进行汇聚的操作；
```

大家可以把Stream当成一个高级版本的Iterator。原始版本的Iterator，
用户只能一个一个的遍历元素并对其执行某些操作；高级版本的Stream，
用户只要给出需要对其包含的元素执行什么操作，
比如"过滤掉长度大于10的字符串"、"获取每个字符串的首字母"等，
具体这些操作如何应用到每个元素上，就给Stream就好了！（这个秘籍，一般人我不告诉他：））
大家看完这些可能对Stream还没有一个直观的认识，莫急，容我慢慢道来！

#### Stream管道

Stream管道包含三部分，缺一不可：

1. stream源

* 零个或多个中间操作
* 一个终止操作，产生一个结果或者一个副作用

```java
int sum = transactions.stream().
        filter(t -> t.getBuyer().getCity().equals("London")).
        mapToInt(Transaction::getPrice).
        sum();
```

transactions.stream() -> stream源

filter/mapToInt -> 中间操作

sum() -> 产生结果

剖析Stream通用语法，再来看一个例子：

```java
//Lists是Guava中的一个工具类
List<Integer> nums = Lists.newArrayList(1,null,3,4,null,6);
nums.stream().filter(num -> num != null).count();
```

![](https://xnstatic-1253397658.file.myqcloud.com/0001.jpg)

图片就是对于Stream例子的一个解析，可以很清楚的看见：原本一条语句被三种颜色的框分割成了三个部分。
红色框中的语句是一个Stream的生命开始的地方，负责创建一个Stream实例；
绿色框中的语句是赋予Stream灵魂的地方，把一个Stream转换成另外一个Stream，
红框的语句生成的是一个包含所有nums变量的Stream，进过绿框的filter方法以后，
重新生成了一个过滤掉原nums列表所有null以后的Stream；
蓝色框中的语句是丰收的地方，把Stream的里面包含的内容按照某种算法来汇聚成一个值，
例子中是获取Stream中包含的元素个数。

#### 使用Stream的基本步骤

1. 创建Stream；
1. 转换Stream，每次转换原有Stream对象不改变，返回一个新的Stream对象（**可以有多次转换**）；
1. 对Stream进行聚合（Reduce）操作，获取想要的结果；

#### stream源

有很多方式可以产生stream源：

1\. 从集合和数组产生：

```java
Collection.stream()  //接口default方法
Collection.parallelStream()  //接口default方法
Arrays.stream(T array) or Stream.of()  // 接口default方法或者是静态方法
```

2\. 静态工厂方法：

```java
IntStream.range()
Files.walk()
```

3\. 使用Stream静态方法来创建Stream源

1) of方法：有两个overload方法，一个接受变长参数，一个接口单一值

```java
Stream<Integer> integerStream = Stream.of(1, 2, 3, 5);
Stream<String> stringStream = Stream.of("taobao");
```

2) generator方法：生成一个无限长度的Stream，
   其元素的生成是通过给定的Supplier（这个接口可以看成一个对象的工厂，每次调用返回一个给定类型的对象）

```java
Stream.generate(new Supplier<Double>() {
    @Override
    public Double get() {
        return Math.random();
    }
});
Stream.generate(() -> Math.random());
Stream.generate(Math::random);
```

三条语句的作用都是一样的，只是使用了lambda表达式和方法引用的语法来简化代码。
每条语句其实都是生成一个无限长度的Stream，其中值是随机的。
这个无限长度Stream是懒加载，一般这种无限长度的Stream都会配合Stream的limit()方法来用。

4\. iterate方法：也是生成无限长度的Stream

和generator不同的是，其元素的生成是重复对给定的种子值(seed)调用用户指定函数来生成的。
其中包含的元素可以认为是：seed，f(seed),f(f(seed))无限循环

```java
Stream.iterate(1, item -> item + 1).limit(10).forEach(System.out::println);
```

这段代码就是先获取一个无限长度的正整数集合的Stream，然后取出前10个打印。千万记住使用limit方法，不然会无限打印下去。

#### 转换Stream

转换Stream其实就是把一个Stream通过某些行为转换成一个新的Stream。Stream接口中定义了几个常用的转换方法，下面我们挑选几个常用的转换方法来解释。

1. distinct: 对于Stream中包含的元素进行去重操作（去重逻辑依赖元素的equals方法），新生成的Stream中没有重复的元素；
1. filter: 对于Stream中包含的元素使用给定的过滤函数进行过滤操作，新生成的Stream只包含符合条件的元素；
1. map: 对于Stream中包含的元素使用给定的转换函数进行转换操作，新生成的Stream只包含转换生成的元素。
   这个方法有三个对于原始类型的变种方法，分别是：mapToInt，mapToLong和mapToDouble。
   这三个方法也比较好理解，比如mapToInt就是把原始Stream转换成一个新的Stream，
   这个新生成的Stream中的元素都是int类型。之所以会有这样三个变种方法，可以免除自动装箱/拆箱的额外消耗；
1. flatMap：和map类似，不同的是其每个元素转换得到的是Stream对象，会把子Stream中的元素压缩到父集合中；
1. peek: 生成一个包含原Stream的所有元素的新Stream，同时会提供一个消费函数（Consumer实例），新Stream每个元素被消费的时候都会执行给定的消费函数；
1. limit: 对一个Stream进行截断操作，获取其前N个元素，如果原Stream中包含的元素个数小于N，那就获取其所有的元素；
1. skip: 返回一个丢弃原Stream的前N个元素后剩下元素组成的新Stream，如果原Stream中包含的元素个数小于N，那么返回空Stream；

#### 性能问题

有些细心的同学可能会有这样的疑问：在对于一个Stream进行多次转换操作，每次都对Stream的每个元素进行转换，
而且是执行多次，这样时间复杂度就是一个for循环里把所有操作都做掉的N（转换的次数）倍啊。其实不是这样的，
转换操作都是lazy的，多个转换操作只会在汇聚操作的时候融合起来，一次循环完成。我们可以这样简单的理解，
Stream里有个操作函数的集合，每次转换操作就是把转换函数放入这个集合中，
在汇聚操作的时候循环Stream对应的集合，然后对每个元素执行所有的函数。

#### 聚集（Reduce）Stream

可变汇聚

可变汇聚对应的只有一个方法：collect，正如其名字显示的，它可以把Stream中的要有元素收集到一个结果容器中（比如Collection）。

通用的collect方法的定义（还有其他override方法）：

```java
<R> R collect(Supplier<R> supplier,
        BiConsumer<R, ? super T> accumulator,
        BiConsumer<R, R> combiner);
```

先来看看这三个参数的含义：Supplier supplier是一个工厂函数，用来生成一个新的容器；
BiConsumer accumulator也是一个函数，用来把Stream中的元素添加到结果容器中；
BiConsumer combiner还是一个函数，用来把中间状态的多个结果容器合并成为一个（并发的时候会用到）

还有好消息，Java8还给我们提供了Collector的工具类–[Collectors]

```java
List<Integer> numsWithoutNull = nums.stream().filter(num -> num != null).
       collect(Collectors.toList());
```

#### 其他汇聚

reduce方法：reduce方法非常的通用，后面介绍的count，sum等都可以使用其实现。
reduce方法有三个override的方法，本文介绍两个最常用的，
先来看reduce方法的第一种形式，其方法定义如下：

```java
Optional<T> reduce(BinaryOperator<T> accumulator);
```

接受一个BinaryOperator类型的参数，在使用的时候我们可以用lambda表达式来。

```java
List<Integer> ints = Lists.newArrayList(1,2,3,4,5,6,7,8,9,10);
System.out.println("ints sum is:" + ints.stream().reduce((sum, item) -> sum + item).get());
```

可以看到reduce方法接受一个函数，这个函数有两个参数，第一个参数是上次函数执行的返回值（也称为中间结果），
第二个参数是stream中的元素，这个函数把这两个值相加，得到的和会被赋值给下次执行这个函数的第一个参数。

reduce方法还有一个很常用的变种：

```java
T reduce(T identity, BinaryOperator<T> accumulator);
```

这个定义了初值，不是默认的第一个位初值。

其他参数:

* allMatch：是不是Stream中的所有元素都满足给定的匹配条件
* anyMatch：Stream中是否存在任何一个元素满足匹配条件
* findFirst: 返回Stream中的第一个元素，如果Stream为空，返回空Optional
* noneMatch：是不是Stream中的所有元素都不满足给定的匹配条件
* max和min：使用给定的比较器（Operator），返回Stream中的最大|最小值

其他Tips：

Optional防止空指针异常，考虑一个常见的嵌套调用：

```java
String version = computer.getSoundcard().getUSB().getVersion();
```

在之前的Java中，我们对于空指针需要这么做：

```java
String version = "UNKNOWN";
if(computer != null){
    Soundcard soundcard = computer.getSoundcard();
    if(soundcard != null){
        USB usb = soundcard.getUSB();
        if(usb != null){
            version = usb.getVersion();
        }
    }
}
```

很显然，这个种做法太挫了！
Groovy语言里面有个?.的语法可以非常优雅的解决这个问题：

```groovy
String version = computer?.getSoundcard()?.getUSB()?.getVersion() ?: "UNKNOWN";
```

当然了，Java8不能示弱啊，所以就有了Optional：

```java
Optional computer = Optional.ofNullable(computer);
String name = computer.map(Computer::getSoundcard)
        .map(Soundcard::getUSB)
        .map(USB::getVersion)
        .orElse("UNKNOWN");
```

关于Opational的更多信息，请参考[Oracle官网]
(http://www.oracle.com/technetwork/articles/java/java8-optional-2175753.html)

### 结束语

1. Java需要lambda表达式和Stream API，充分发挥多核并行的优势，大大提高核心库的运行速度。
1. 通过default关键字扩展接口来进行接口演变，同时保持向后兼容。
1. 通过lambda表达式，大大简化了集合类的操作
1. Java8同时在语言、虚拟机、核心库方面做了大幅度的改进和优化，使得编程更简单，更快速。
