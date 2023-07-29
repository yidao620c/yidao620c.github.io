---
title: Java泛型的PECS原则
date: '2018-05-30 12:22:11 +0800'
toc: false
categories: [ java ]
tags: [ java ]
---

在泛型编程时，使用部分限定的形参时，`<? super T>`和`<? extends T>`的使用场景容易混淆，
PECS原则可以帮助我们很好记住它们：提供者（Provider）使用extends，消费者（Consumer）使用super。

> [!NOTE]
> Provider指的就是该容器从自己的容器里提供T类型或T的子类型的对象供别人使用；
> 
> Consumer指的就是该容器把从别处拿到的T类型或T的子类型的对象放到自己的容器。

<!-- more -->

## 泛型擦除

要理解 `super` 和 `extends` 的边界问题，首先要理解泛型擦除。

这里我们先定义一组继承关系的类，以水果家族为例，定义下面的水果、苹果、红苹果、橘子。

``` java
class Fruit {
}

class Apple extends Fruit {
}

class RedApple extends Apple {
}

class Orange extends Fruit {
}
```

然后测试一下泛型在运行时的类型
``` java
public static void testClassInfo() {
    Class a = new ArrayList<Apple>().getClass();
    Class b = new ArrayList<Orange>().getClass();
    System.out.println(a == b);
}

public static void main(String[] args) {
    testClassInfo();
}
```

打印结果为true。

因为在泛型代码内部，无法获取任何有关泛型参数类型的任何信息！Java的泛型就是使用擦除来实现的，
当你在使用泛型的时候，任何信息都被擦除，你所知道的就是你在使用一个对象。
所以 `List<Apple>` 和 `List<Orange>` 在运行时，会被擦除成他们的原生类型List。

泛型不能用于显性地引用运行时类型的操作之中，例如 转型，`instanceof` 和 `new` 操作（包括 new一个对象，new一个数组），
因为所有关于参数的类型信息都在运行时丢失了，所以任何在运行时需要获取类型信息的操作都无法进行工作。

## 上界PE原则

``` java
List<Apple> apples = new ArrayList<>();
List<Fruit> fruits = apples;  // 编译不过。
```

上边的代码定义编译是通不过的，因为虽然`Fruit`是`Apple`的父类，但是`List<Fruit>`并不是`List<Apple>`的父类，
如果想实现类似的效果，可以用如下方式定义：

``` java
List<Apple> apples = new ArrayList<>();
List<? extends Fruit> fruits = apples; // √ 编译通过。
```

说明，`List<Fruit>`不是`List<Apple>`的父类，`List<? extends Fruit>`才是！

实际上，`List<? extends Fruit>`是`List<T>`的父类！T代表`Fruit`或`Fruit`的任一子类。

上述代码的fruits容器是不能继续执行add方法的，即`<? extends T>` 只能作为provider，执行get方法。
因为如果能够执行add方法，就会造成fruits中的对象不能确定类型，如果放进去的是`Apple`还好，因为类型相同；
但是如果放入的是`Orange`，其实就是相当于执行了`apples.add(Orange orange)`，肯定是不行的。所以禁止add。

## 下界CS原则

按照上边的说法类比：`List<? super Apple>`是`List<T>`的父类！T为`Apple`类或其任一父类(注意此处为父类，PE中是子类)。

``` java
public static void testSuper() {
    List<? super Apple> list = new ArrayList<>();
    list.add(new Apple());
    list.add(new RedApple());
    list.add(new Fruit()); // 编译不通过

    Object obj = list.get(0);
}
```

这里可以看到，`list.add(new Fruit())` 这句不能编译成功，这是因为 `List<? super Apple>` 
表示`具有Apple的父类的列表`。但是编译器不知道你要添加哪种`Apple`的父类，因此不能安全地添加。
而为啥能添加Apple或者它的子类呢？因为容器能接受的是Apple或者它的父类，那咋样都能安全向上转换成它的父类的。

`List<? super T>`类型的集合是不允许get的，但是你找不到一个合适的类型来声明其引用，因为T类型的父类型可能不只一个，
到底是哪一个不能确定。如果实在是要get，则只能返回Object类型。

## PECS的典型使用场景

下面是一个列表元素复制的例子

``` java
import java.util.Arrays;
import java.util.Collections;
import java.util.List;

public class PecsTest {
    public static void main(String[] args) {
        List<String> src = Arrays.asList("aaa", "bbb", "ccc", "ddd");
        List<String> dest = Arrays.asList(new String[src.size()]);
        Collections.copy(dest, src);
        System.out.println(dest);
    }
}
```

下面是`Collections.copy`方法的源码
``` java
public static <T> void copy(List<? super T> dest, List<? extends T> src) {
    int srcSize = src.size();
    if (srcSize > dest.size())
        throw new IndexOutOfBoundsException("Source does not fit in dest");

    if (srcSize < COPY_THRESHOLD ||
        (src instanceof RandomAccess && dest instanceof RandomAccess)) {
        for (int i=0; i<srcSize; i++)
            dest.set(i, src.get(i));
    } else {
        ListIterator<? super T> di=dest.listIterator();
        ListIterator<? extends T> si=src.listIterator();
        for (int i=0; i<srcSize; i++) {
            di.next();
            di.set(si.next());
        }
    }
}
```

这样我们总结super和extends的使用，得到一个更广泛的原则。

* `super`用来限制传入的参数
* `extends`使用限制返回的参数

## MappingFunction例子
再来看一个典型的例子。在最新的ConcurrentHashMap中有这么一个方法声明
``` java
@SuppressWarnings("unchecked")
public V computeIfAbsent(K key, MappingFunction<? super K, ? extends V> mappingFunction) {
    if (key == null || mappingFunction == null)
        throw new NullPointerException();
    return (V)internalComputeIfAbsent(key, mappingFunction);
}
```

`MappingFunction<? super K, ? extends V>` 的泛型声明吸引了我。

MappingFunction接口定义如下
``` java
public static interface MappingFunction<K, V> {
    /**
     * Returns a non-null value for the given key.
     *
     * @param key the (non-null) key
     * @return a non-null value
     */
    V map(K key);
}
```

从上面的map中可见，`MappingFunction<K,V>`中的K在`computeIfAbsent`中声明ConcurrentHashMap<K,V>的K为其super边界，
V声明ConcurrentHashMap<K,V>的V为其extends边界。即map方法可以接收ConcurrentHashMap key的父类(包括边界)，
返回的value必须是ConcurrentHashMap value的子类(包括边界)。

假如有`ConcurrentHashMap<Apple, Fruit>`, 那么方法`computeIfAbsent`可以接受`MappingFunction<Object, Orange>`。
`Orange map(Object)` 的方法可以这样使用 `Fruit result =  map(Apple)`