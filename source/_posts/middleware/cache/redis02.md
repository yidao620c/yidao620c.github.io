---
title: redis笔记02 - 基本操作
date: 2015-07-06 16:15:42 +0800
toc: false
categories: [中间件]
tags: [ redis ]
---

strings类型及操作

string是最简单的类型，你可以理解成与Memcached是一模一样的类型，一个key对应一个value，
其上支持的操作与Memcached的操作类似。但它的功能更丰富。

string类型是二进制安全的。意思是redis的string可以包含任何数据，比如jpg图片或者序列化的对象。
从内部实现来看其实string可以看作byte数组，最大上限是1G字节，下面是string类型的定义:

```c
struct sdshdr {
　　long len;
　　long free;
　　char buf[];
};
```

len是buf数组的长度。
<!-- more -->

free是数组中剩余可用字节数，由此可以理解为什么string类型是二进制安全的了，因为它本质上就是个byte数组，当然可以包含任何数据了

buf是个char数组用于存贮实际的字符串内容，其实char和c#中的byte是等价的，都是一个字节。

另外string类型可以被部分命令按int处理.比如incr等命令，如果只用string类型，redis就可以被看作加上持久化特性的memcached。
当然redis对string类型的操作比memcached还是多很多的，具体操作方法如下：

1、set

设置key对应的值为string类型的value。
例如我们添加一个name= HongWan的键值对，可以这样做:

```bash
redis 127.0.0.1:6379> set name HongWan
OK
redis 127.0.0.1:6379>
```

2、setnx

设置key对应的值为string类型的value。如果key已经存在，返回0，nx是not exist的意思。
例如我们添加一个name= HongWan_new的键值对，可以这样做:

```bash
redis 127.0.0.1:6379> get name
"HongWan"
redis 127.0.0.1:6379> setnx name HongWan_new
(integer) 0
redis 127.0.0.1:6379> get name
"HongWan"
redis 127.0.0.1:6379>
```

由于原来name有一个对应的值，所以本次的修改不生效，且返回码是0。

3、setex

设置key对应的值为string类型的value，并指定此键值对应的有效期。
例如我们添加一个haircolor= red的键值对，并指定它的有效期是10秒，可以这样做:

```bash
redis 127.0.0.1:6379> setex haircolor 10 red
OK
redis 127.0.0.1:6379> get haircolor
"red"
redis 127.0.0.1:6379> get haircolor
(nil)
redis 127.0.0.1:6379>
```

可见由于最后一次的调用是10秒以后了，所以取不到haicolor这个键对应的值。

4、setrange

设置指定key的value值的子字符串。
例如我们希望将HongWan的126邮箱替换为gmail邮箱，那么我们可以这样做:

```bash
redis 127.0.0.1:6379> get name
"HongWan@126.com"
redis 127.0.0.1:6379> setrange name 8 gmail.com
(integer) 17
redis 127.0.0.1:6379> get name
"HongWan@gmail.com"
redis 127.0.0.1:6379>
```

其中的8是指从下标为8(包含8)的字符开始替换

5、mset

一次设置多个key的值，成功返回ok表示所有的值都设置了，失败返回0表示没有任何值被设置。

```bash
redis 127.0.0.1:6379> mset key1 HongWan1 key2 HongWan2
OK
redis 127.0.0.1:6379> get key1
"HongWan1"
redis 127.0.0.1:6379> get key2
"HongWan2"
redis 127.0.0.1:6379>
```

6、msetnx

一次设置多个key的值，成功返回ok表示所有的值都设置了，失败返回0表示没有任何值被设置，但是不会覆盖已经存在的key。

```bash
redis 127.0.0.1:6379> get key1
"HongWan1"
redis 127.0.0.1:6379> get key2
"HongWan2"
redis 127.0.0.1:6379> msetnx key2 HongWan2_new key3 HongWan3
(integer) 0
redis 127.0.0.1:6379> get key2
"HongWan2"
redis 127.0.0.1:6379> get key3
(nil)
```

可以看出如果这条命令返回0，那么里面操作都会回滚，都不会被执行。

7、get

获取key对应的string值,如果key不存在返回nil。
例如我们获取一个库中存在的键name，可以很快得到它对应的value

```bash
redis 127.0.0.1:6379> get name
"HongWan"
redis 127.0.0.1:6379>
我们获取一个库中不存在的键name1，那么它会返回一个nil以表时无此键值对
redis 127.0.0.1:6379> get name1
(nil)
redis 127.0.0.1:6379>
```

8、getset

设置key的值，并返回key的旧值。

```bash
redis 127.0.0.1:6379> get name
"HongWan"
redis 127.0.0.1:6379> getset name HongWan_new
"HongWan"
redis 127.0.0.1:6379> get name
"HongWan_new"
redis 127.0.0.1:6379>
```

接下来我们看一下如果key不存的时候会什么样儿?

```bash
redis 127.0.0.1:6379> getset name1 aaa
(nil)
redis 127.0.0.1:6379>
```

可见，如果key不存在，那么将返回nil

9、getrange

获取指定key的value值的子字符串。 具体样例如下:

```bash
redis 127.0.0.1:6379> get name
"HongWan@126.com"
redis 127.0.0.1:6379> getrange name 0 6
"HongWan"
redis 127.0.0.1:6379>
```

字符串左面下标是从0开始的

```bash
redis 127.0.0.1:6379> getrange name -7 -1
"126.com"
redis 127.0.0.1:6379>
```

字符串右面下标是从-1开始的

```bash
redis 127.0.0.1:6379> getrange name 7 100
"@126.com"
redis 127.0.0.1:6379>
```

当下标超出字符串长度时，将默认为是同方向的最大下标

10、mget

一次获取多个key的值，如果对应key不存在，则对应返回nil。
具体样例如下:

```bash
redis 127.0.0.1:6379> mget key1 key2 key3
1) "HongWan1"
2) "HongWan2"
3) (nil)
redis 127.0.0.1:6379>
```

key3由于没有这个键定义，所以返回nil。

11、incr

对key的值做加加操作,并返回新的值。注意incr一个不是int的value会返回错误，incr一个不存在的key，则设置key为1

```bash
redis 127.0.0.1:6379> set age 20
OK
redis 127.0.0.1:6379> incr age
(integer) 21
redis 127.0.0.1:6379> get age
"21"
redis 127.0.0.1:6379>
```

12、incrby

同incr类似，加指定值 ，key不存在时候会设置key，并认为原来的value是 0

```bash
redis 127.0.0.1:6379> get age
"21"
redis 127.0.0.1:6379> incrby age 5
(integer) 26
redis 127.0.0.1:6379> get name
"HongWan@gmail.com"
redis 127.0.0.1:6379> get age
"26"
redis 127.0.0.1:6379>
```

13、decr

对key的值做的是减减操作，decr一个不存在key，则设置key为-1

```bash
redis 127.0.0.1:6379> get age
"26"
redis 127.0.0.1:6379> decr age
(integer) 25
redis 127.0.0.1:6379> get age
"25"
redis 127.0.0.1:6379>
```

14、decrby

同decr，减指定值。

```bash
redis 127.0.0.1:6379> get age
"25"
redis 127.0.0.1:6379> decrby age 5
(integer) 20
redis 127.0.0.1:6379> get age
"20"
redis 127.0.0.1:6379>
```

decrby完全是为了可读性，我们完全可以通过incrby一个负值来实现同样效果，反之一样。

```bash
redis 127.0.0.1:6379> get age
"20"
redis 127.0.0.1:6379> incrby age -5
(integer) 15
redis 127.0.0.1:6379> get age
"15"
redis 127.0.0.1:6379>
```

15、append

给指定key的字符串值追加value,返回新字符串值的长度。例如我们向name的值追加一个@126.com字符串，那么可以这样做:

```bash
redis 127.0.0.1:6379> append name @126.com
(integer) 15
redis 127.0.0.1:6379> get name
"HongWan@126.com"
redis 127.0.0.1:6379>
```

16、strlen

取指定key的value值的长度。

```bash
redis 127.0.0.1:6379> get name
"HongWan_new"
redis 127.0.0.1:6379> strlen name
(integer) 11
redis 127.0.0.1:6379> get age
"15"
redis 127.0.0.1:6379> strlen age
(integer) 2
redis 127.0.0.1:6379>
```

2）hash

Redis hash是一个string类型的field和value的映射表.它的添加、删除操作都是O(1)(平均)
。hash特别适合用于存储对象。相较于将对象的每个字段存成单个string类型。将一个对象存储在hash类型中会占用更少的内存，并且可以更方便的存取整个对象。省内存的原因是新建一个hash对象时开始是用zipmap(
又称为small hash)来存储的。这个zipmap其实并不是hash
table，但是zipmap相比正常的hash实现可以节省不少hash本身需要的一些元数据存储开销。尽管zipmap的添加，删除，查找都是O(n)
，但是由于一般对象的field数量都不太多。所以使用zipmap也是很快的,也就是说添加删除平均还是O(1)
。如果field或者value的大小超出一定限制后，Redis会在内部自动将zipmap替换成正常的hash实现. 这个限制可以在配置文件中指定

```
hash-max-zipmap-entries 64 #配置字段最多64个。
hash-max-zipmap-value 512 #配置value最大为512字节。
```

1、hset

设置hash field为指定值，如果key不存在，则先创建。

```bash
redis 127.0.0.1:6379> hset myhash field1 Hello
(integer) 1
redis 127.0.0.1:6379>
```

2、hsetnx

设置hash field为指定值，如果key不存在，则先创建。如果field已经存在，返回0，nx是not exist的意思。

```bash
redis 127.0.0.1:6379> hsetnx myhash field "Hello"
(integer) 1
redis 127.0.0.1:6379> hsetnx myhash field "Hello"
(integer) 0
redis 127.0.0.1:6379>
```

第一次执行是成功的，但第二次执行相同的命令失败，原因是field已经存在了。

3、hmset

同时设置hash的多个field。

```bash
redis 127.0.0.1:6379> hmset myhash field1 Hello field2 World
OK
redis 127.0.0.1:6379>
```

4、hget

获取指定的hash field。

```bash
redis 127.0.0.1:6379> hget myhash field1
"Hello"
redis 127.0.0.1:6379> hget myhash field2
"World"
redis 127.0.0.1:6379> hget myhash field3
(nil)
redis 127.0.0.1:6379>
```

由于数据库没有field3，所以取到的是一个空值nil。

5、hmget

获取全部指定的hash filed。

```bash
redis 127.0.0.1:6379> hmget myhash field1 field2 field3
1) "Hello"
2) "World"
3) (nil)
redis 127.0.0.1:6379>
```

由于数据库没有field3，所以取到的是一个空值nil。

6、hincrby

指定的hash filed 加上给定值。

```bash
redis 127.0.0.1:6379> hset myhash field3 20
(integer) 1
redis 127.0.0.1:6379> hget myhash field3
"20"
redis 127.0.0.1:6379> hincrby myhash field3 -8
(integer) 12
redis 127.0.0.1:6379> hget myhash field3
"12"
redis 127.0.0.1:6379>
```

在本例中我们将field3的值从20降到了12，即做了一个减8的操作。

7、hexists

测试指定field是否存在。

```bash
redis 127.0.0.1:6379> hexists myhash field1
(integer) 1
redis 127.0.0.1:6379> hexists myhash field9
(integer) 0
redis 127.0.0.1:6379>
```

通过上例可以说明field1存在，但field9是不存在的。

8、hlen

返回指定hash的field数量。

```bash
redis 127.0.0.1:6379> hlen myhash
(integer) 4
redis 127.0.0.1:6379>
```

通过上例可以看到myhash中有4个field。

9、hdel

返回指定hash的field数量。

```bash
redis 127.0.0.1:6379> hlen myhash
(integer) 4
redis 127.0.0.1:6379> hdel myhash field1
(integer) 1
redis 127.0.0.1:6379> hlen myhash
(integer) 3
redis 127.0.0.1:6379>
```

10、hkeys

返回hash的所有field。

```bash
redis 127.0.0.1:6379> hkeys myhash
1) "field2"
2) "field"
3) "field3"
redis 127.0.0.1:6379>
```

说明这个hash中有3个field。

11、hvals

返回hash的所有value。

```bash
redis 127.0.0.1:6379> hvals myhash
1) "World"
2) "Hello"
3) "12"
redis 127.0.0.1:6379>
```

说明这个hash中有3个field。

12、hgetall

获取某个hash中全部的filed及value。

```bash
redis 127.0.0.1:6379> hgetall myhash
1) "field2"
2) "World"
3) "field"
4) "Hello"
5) "field3"
6) "12"
redis 127.0.0.1:6379>
```

可见，一下子将myhash中所有的field及对应的value都取出来了。

3）list

Redis的list类型其实就是一个每个子元素都是string类型的双向链表。链表的最大长度是(2的32次方)
。我们可以通过push,pop操作从链表的头部或者尾部添加删除元素。这使得list既可以用作栈，也可以用作队列。

有意思的是list的pop操作还有阻塞版本的，当我们[lr]
pop一个list对象时，如果list是空，或者不存在，会立即返回nil。但是阻塞版本的b[lr]
pop可以则可以阻塞，当然可以加超时时间，超时后也会返回nil。为什么要阻塞版本的pop呢，主要是为了避免轮询。举个简单的例子如果我们用list来实现一个工作队列。执行任务的thread可以调用阻塞版本的pop去获取任务这样就可以避免轮询去检查是否有任务存在。当任务来时候工作线程可以立即返回，也可以避免轮询带来的延迟。说了这么多，接下来看一下实际操作的方法吧：

1、lpush

在key对应list的头部添加字符串元素：

```bash
redis 127.0.0.1:6379> lpush mylist "world"
(integer) 1
redis 127.0.0.1:6379> lpush mylist "hello"
(integer) 2
redis 127.0.0.1:6379> lrange mylist 0 -1
1) "hello"
2) "world"
redis 127.0.0.1:6379>
```

在此处我们先插入了一个world，然后在world的头部插入了一个hello。其中lrange是用于取mylist的内容。

2、rpush

在key对应list的尾部添加字符串元素：

```bash
redis 127.0.0.1:6379> rpush mylist2 "hello"
(integer) 1
redis 127.0.0.1:6379> rpush mylist2 "world"
(integer) 2
redis 127.0.0.1:6379> lrange mylist2 0 -1
1) "hello"
2) "world"
redis 127.0.0.1:6379>
```

在此处我们先插入了一个hello，然后在hello的尾部插入了一个world。

3、linsert

在key对应list的特定位置之前或之后添加字符串元素：

```bash
redis 127.0.0.1:6379> rpush mylist3 "hello"
(integer) 1
redis 127.0.0.1:6379> rpush mylist3 "world"
(integer) 2
redis 127.0.0.1:6379> linsert mylist3 before "world" "there"
(integer) 3
redis 127.0.0.1:6379> lrange mylist3 0 -1
1) "hello"
2) "there"
3) "world"
redis 127.0.0.1:6379>
```

在此处我们先插入了一个hello，然后在hello的尾部插入了一个world，然后又在world的前面插入了there。

4、lset

设置list中指定下标的元素值(下标从0开始)：

```bash
redis 127.0.0.1:6379> rpush mylist4 "one"
(integer) 1
redis 127.0.0.1:6379> rpush mylist4 "two"
(integer) 2
redis 127.0.0.1:6379> rpush mylist4 "three"
(integer) 3
redis 127.0.0.1:6379> lset mylist4 0 "four"
OK
redis 127.0.0.1:6379> lset mylist4 -2 "five"
OK
redis 127.0.0.1:6379> lrange mylist4 0 -1
1) "four"
2) "five"
3) "three"
redis 127.0.0.1:6379>
```

在此处我们依次插入了one,two,three，然后将标是0的值设置为four，再将下标是-2的值设置为five。

5、lrem

从key对应list中删除count个和value相同的元素。
count>0时，按从头到尾的顺序删除，具体如下：

```bash
redis 127.0.0.1:6379> rpush mylist5 "hello"
(integer) 1
redis 127.0.0.1:6379> rpush mylist5 "hello"
(integer) 2
redis 127.0.0.1:6379> rpush mylist5 "foo"
(integer) 3
redis 127.0.0.1:6379> rpush mylist5 "hello"
(integer) 4
redis 127.0.0.1:6379> lrem mylist5 2 "hello"
(integer) 2
redis 127.0.0.1:6379> lrange mylist5 0 -1
1) "foo"
2) "hello"
redis 127.0.0.1:6379>
```

count<0时，按从尾到头的顺序删除，具体如下：

```bash
redis 127.0.0.1:6379> rpush mylist6 "hello"
(integer) 1
redis 127.0.0.1:6379> rpush mylist6 "hello"
(integer) 2
redis 127.0.0.1:6379> rpush mylist6 "foo"
(integer) 3
redis 127.0.0.1:6379> rpush mylist6 "hello"
(integer) 4
redis 127.0.0.1:6379> lrem mylist6 -2 "hello"
(integer) 2
redis 127.0.0.1:6379> lrange mylist6 0 -1
1) "hello"
2) "foo"
redis 127.0.0.1:6379>
```

count=0时，删除全部，具体如下：

```bash
redis 127.0.0.1:6379> rpush mylist7 "hello"
(integer) 1
redis 127.0.0.1:6379> rpush mylist7 "hello"
(integer) 2
redis 127.0.0.1:6379> rpush mylist7 "foo"
(integer) 3
redis 127.0.0.1:6379> rpush mylist7 "hello"
(integer) 4
redis 127.0.0.1:6379> lrem mylist7 0 "hello"
(integer) 3
redis 127.0.0.1:6379> lrange mylist7 0 -1
1) "foo"
redis 127.0.0.1:6379>
```

6、ltrim

保留指定key 的值范围内的数据：

```bash
redis 127.0.0.1:6379> rpush mylist8 "one"
(integer) 1
redis 127.0.0.1:6379> rpush mylist8 "two"
(integer) 2
redis 127.0.0.1:6379> rpush mylist8 "three"
(integer) 3
redis 127.0.0.1:6379> rpush mylist8 "four"
(integer) 4
redis 127.0.0.1:6379> ltrim mylist8 1 -1
OK
redis 127.0.0.1:6379> lrange mylist8 0 -1
1) "two"
2) "three"
3) "four"
redis 127.0.0.1:6379>
```

7、lpop

从list的头部删除元素，并返回删除元素：

```bash
redis 127.0.0.1:6379> lrange mylist 0 -1
1) "hello"
2) "world"
redis 127.0.0.1:6379> lpop mylist
"hello"
redis 127.0.0.1:6379> lrange mylist 0 -1
1) "world"
redis 127.0.0.1:6379>
```

8、rpop

从list的尾部删除元素，并返回删除元素：

```bash
redis 127.0.0.1:6379> lrange mylist2 0 -1
1) "hello"
2) "world"
redis 127.0.0.1:6379> rpop mylist2
"world"
redis 127.0.0.1:6379> lrange mylist2 0 -1
1) "hello"
redis 127.0.0.1:6379>
```

9、rpoplpush

从第一个list的尾部移除元素并添加到第二个list的头部,最后返回被移除的元素值，整个操作是原子的.如果第一个list是空或者不存在返回nil：

```bash
redis 127.0.0.1:6379> lrange mylist5 0 -1
1) "three"
2) "foo"
3) "hello"
redis 127.0.0.1:6379> lrange mylist6 0 -1
1) "hello"
2) "foo"
redis 127.0.0.1:6379> rpoplpush mylist5 mylist6
"hello"
redis 127.0.0.1:6379> lrange mylist5 0 -1
1) "three"
2) "foo"
redis 127.0.0.1:6379> lrange mylist6 0 -1
1) "hello"
2) "hello"
3) "foo"
redis 127.0.0.1:6379>
```

10、lindex

返回名称为key的list中index位置的元素：

```bash
redis 127.0.0.1:6379> lrange mylist5 0 -1
1) "three"
2) "foo"
redis 127.0.0.1:6379> lindex mylist5 0
"three"
redis 127.0.0.1:6379> lindex mylist5 1
"foo"
redis 127.0.0.1:6379>
```

11、llen

返回key对应list的长度：

```bash
redis 127.0.0.1:6379> llen mylist5
(integer) 2
redis 127.0.0.1:6379>
```

