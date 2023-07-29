---
title: python核心 - 生成器
date: 2015-12-02 22:12:42 +0800
toc: true
categories: [ python ]
tags: [ python核心 ]
---

在讲生成器之前，先讲讲python里面常用的几个常见的推导式：

*列表推导式（list comprehension）*

```python
my_list = [f(x) for x in sequence if cond(x)]
```

*字典推导式（dictionary comprehension）*

```python
my_dict = {k(x): v(x) for x in sequence if cond(x)}
```

*集合推导式（set comprehension）*

```python
my_set = {f(x) for x in sequence if cond(x)}
```

*生成器表达式（generator expression）*

```python
my_generator = (f(x) for x in sequence if cond(x))
```

对于大部分的python初学者而言，生成器和`yield`关键字比较难以理解。很多文章解释的不清楚，
这篇文章我想深入的讲解这个`yield`关键字，它到底是个什么东西，为什么它如此的重要，以及我们该如何去使用它。

注意：近些年来，随着越来越多的特性被加入到PEP中，生成器变得越来越强大。我在下一篇文章会深入讲解协程、多任务协作和异步I/O这些高阶知识，来见识下`yield`
的威力。
<!-- more -->

### yield关键字

当我们调用一个普通的函数时，执行过程从第一条语句开始，直到碰到一个`return`语句或者遇到一个异常抛出，
再或者到了函数最后一条语句(实际上相对于一个隐式的`return None`)的时候结束。
一旦这个函数返回后将控制权交还给它的调用者，它里面所有的局部变量值都消失了，当你重新调用它的时候，一切又将重新开始。

这就是我们通常意义上面所认识的函数（或者说是子程序），但有时候我们需要创建某个函数，它并不简单的返回一个值，
而是可以不断的释放一个值的序列。那么这个特殊的函数就需要能够"保存"它的状态。

我刚刚用了"释放一个值的序列"这种说法，是因为我刚刚所假定的那个函数并不是普通的返回一个值。
`return`表明函数将控制权返回给被调用时的那个点。而`yield`表示这个控制权的转义只是暂时的，将来我还可以重获控制权。

在Python中，有这种能力的"函数"被称为生成器，它们相当有用。生成器（yield语句）刚开始被引入进来主要是用来方便的生成序列值。

为了更好的理解生成器的作用，我们先来看一个例子，看看怎样来生成一个序列。

注意：很多时候我会将生成器叫做协程，后面我会用这个术语，所有的协程其实就是生成器，Python官方只定义了生成器这个名字，协程是另一种叫法。

### 有趣的素数

假设你老板叫你写一个函数，接受一个整数列表为参数，然后输出一个只包含素数的Iterable（一次返回一个成员的对象）。

这个很简单嘛

```python
def get_primes(input_list):
    result_list = list()
    for element in input_list:
        if is_prime(element):
            result_list.append()

    return result_list

# or better yet...

def get_primes(input_list):
    return (element for element in input_list if is_prime(element))

# not germane to the example, but here's a possible implementation of
def is_prime(number):
    if number > 1:
        if number == 2:
            return True
        if number % 2 == 0:
            return False
        for current in range(3, int(math.sqrt(number) + 1), 2):
            if number % current == 0:
                return False
        return True
    return False
```

上面任何一个`get_primes()`都能满足要求，所以我们告诉老板任务完成了。老板测试后觉得很满意，符合她的期望。

### 处理无限序列

不过麻烦又来了，过了几天我们老板又跑来了，说她要遇到个小麻烦，她想这个`get_primes()`能处理超大的列表，这个列表太多以至于直接在内存中初始化会撑爆内存。
所以她希望用一个起始值`start`
来调用这个函数，然后返回所有的比它大的素数。（可能她要尝试去解决[欧拉猜想](http://projecteuler.net/problem=10)）

这时候我们得重新设计下`get_primes()`函数了，很明显我们不可能直接返回一个无限的列表（但是又经常能碰到需要操作无限列表的情况）。普通函数也许是行不通了

在我们放弃之前，让我们思考一下这里面最大的障碍是什么。其实就是普通函数只有一次机会返回结果，因此我们必须把所有结果汇总后返回。但有什么办法呢？函数只能一次返回啊。

要是有特殊函数不是这样的呢，设想下如果`get_primes()`
函数一次只返回下一个值，我们就根本不需要构建列表了。没有列表也就占用不了内存，而老板只是想在结果上面进行迭代操作，她并不在意你以什么形式返回来。

不幸的是，这个看上去好像不可能实现，就算我们有个魔法函数允许我们从n循环到无穷大

```python
def get_primes(start):
    for element in magical_infinite_range(start):
        if is_prime(element):
            return element
```

然后我们像下面这样调用这个函数：

```python
def solve_number_10():
    total = 2
    for next_prime in get_primes(3):
        if next_prime < 2000000:
            total += next_prime
        else:
            print(total)
            return
```

很明显，`get_primes()`函数遇到3的时候就返回了，在第4行，但是第二次调用的时候，它又返回了3。
我们需要一种机制，下一次再调用它的时候，能从第4行开始获取下一个数，但是你再次调用函数时，它又重新开始运行，因为函数只有一个入口，就是第一行。

### 走进生成器

这类问题非常普遍，Python于是引入了新的一种机制来解决它：生成器。一个生成器负责生产数据，可以通过简单的调用生成器函数来创建一个生成器。

一个生成器函数定义跟普通函数类似，唯一区别是，它通过`yield`关键字生产数据，而不是用`return`关键字返回。
如果在一个`def`函数定义内有出现了一个`yield`关键字，那么这个函数就变成了生成器函数（就算它还包含return语句）。

生成器函数可创建生成器迭代器，其实就是我们通常所说的生成器。请记住生成器是一类特殊的迭代器。要成为一个迭代器，生成器就必须定义一些特殊方法，其中一个就是`__next__()`，
要从生成器中获取下一个元素，我们使用同样的内置函数`next()`

因此，当你使用`next()`调用一个生成器的时候，生成器负责传会下一个值，它会把`yield`后面的值传过来。所以记住这句话：`yield`
相对于生成器函数的`return`

下面是一个简单的生成器函数：

```python
>> >

def simple_generator_function():
    >> > yield 1

>> > yield 2
>> > yield 3
```

两种使用方法：

```python
>> for value in simple_generator_function():
    >> > print(value)
1
2
3
>> > our_generator = simple_generator_function()
>> > next(our_generator)
1
>> > next(our_generator)
2
>> > next(our_generator)
3
```

### 神奇的背后

那么这是怎么一回事呢。当生成器函数调用`yield`
语句的时候，生成器函数的状态被冻结，所有变量值都被保存，下一条要执行的语句被记录下来，直到下一次的`next()`调用来临。
一旦再遇到`next()`调用，生成器函数就被激活，而如果`next()`永远不再调用，那么最后记录的状态也就被丢弃了。

我们使用生成器函数重新改写下`get_primes()`函数，这次我们不再需要`magical_infinite_range()`函数了。直接使用`while`
循环来创建我们的无限序列：

```python
def get_primes(number):
    while True:
        if is_prime(number):
            yield number
        number += 1
```

如果一个生成器函数调用了`return`语句或者到了最后一条语句，就会抛出一个`StopIteration`异常，它表明不能再调用`next()`
了。因此我们使用`while`循环就能保证它永远不会达到最后，
所以只要我们不断的调用`next()`，它就能源源不断的为我们产生新的值。这也是我们在处理无限序列时常用的做法。

### 程序控制流

我们重新来看看代码`solve_number_10`，它在调用`get_primes()`时究竟发生了什么：

```python
def solve_number_10():
    total = 2
    for next_prime in get_primes(3):
        if next_prime < 2000000:
            total += next_prime
        else:
            print(total)
            return
```

我们一步步来，当我们使用for循环调用`get_primes(3)`时候，其实就是不断的使用`next()`调用它。程序控制流进入`get_primes()`
，我们来分析下这个函数执行流

1. 第3行我们碰到了while循环语句
2. `if`条件判断碰到数字3（是一个素数满足条件）
3. `yield`数字3并将控制权返回给`solve_number_10`

然后，我们再去看`solve_number_10`：

1. 数字3从for循环中拿到
1. for循环将`next_prime`设置成3
1. `next_prime`被加到`total`上
1. for循环再次从`get_primes`请求数据

这时候，控制不会直接从`get_prime()`函数第1行语句进入，而是从第5行恢复运行，那也是我们离开的地方。

```python
def get_primes(number):
    while True:
        if is_prime(number):
            yield number
        number += 1  # <<<<<<<<<<
```

最重要的是，number的值还保存着之前的3。请记住，`yield`会将数字传递给`next()`调用者，并同时保存生成器函数的状态，包括它所有内部的局部变量值。
这时候，number的值+1变成4，然后重新while循环，我们不断的增加这个number的值直到它是一个素数5，这时候，`yield`
将这个5传递给`solve_number_10`中的`for`循环。
这个过程不断的重复，直到for循环停止了（遇到素数大于2000000）。

### 更强大的生成器

在[PEP 342](http://www.python.org/dev/peps/pep-0342/)中，我们可以向生成器传递数据，它让生成器有了三种能力：

1. 可以生产一个数（前面讲过）
1. 接受一个数，这个数是任意的
1. 在一条语句中生产一个数并同时接受另一个数

为了说明怎样向生成器发送数据，还是刚刚那个素数例子，这次我们并不是简单的返回比`number`大的素数，而是找出大于这个数的连续指数最小的素数。
比如对于10，我们要找出比10大的最小素数，然后是比100大的最小素数，然后是比1000大的最小素数，以此类推。

```python
def get_primes(number):
    while True:
        if is_prime(number):
            number = yield number
        number += 1
```

解释下`get_primes()`中的`number = yield number`这句话，它表示，把`number`
发送出去，然后当有人向我发送值的时候就把这个值设置成左边的`number`。
这样我们就能每次进入`get_prime()`下次执行时，将number设置成我们想要的值，而不是离开时它自己保存的值。

然后我们再来改写调用它的函数：

```python
def print_successive_primes(iterations, base=10):
    prime_generator = get_primes(base)
    prime_generator.send(None)
    for power in range(iterations):
        print(prime_generator.send(base ** power))

```

两件事需要说明下：

1. 我们打印`generator.send`的结果，因为`send`做了两件事，一是给生成器发送数据，并同时返回生成器生产的值。
2. 注意到这一行`prime_generator.send(None)`，当我们使用`send`
   来启动一个生成器（也就是从生成器第一行开始执行一直到碰到第一个`yield`语句），你必须发送`None`。
   这个是有必要的，因为生成器函数定义的第一条语句并不是yield，因此要是我们发送一个真实的数据过去，没东西可以接受啊。一旦这个生成器启动了，我们就能给它发送数据了。

在来看一个send例子

```python
if __name__ == '__main__':
    def myfunc():
        print
        'start myfunc...',
        m = yield 111
        print
        'get m = %s' % m
        n = yield 222
        print
        'get n = %s' % n
        yield


    def caller():
        c = myfunc()  # 调用生成器函数获取到生成器c
        result = c.next()  # 相当于c.send(None)
        print
        '(1) result=%s' % result
        result = c.send('Hello')
        print
        '(2) result=%s' % result
        c.send('Goodbye')


    caller()
```

我用图片方式标注几个重要的执行点，然后分析一下整个执行流程：
![](https://xnstatic-1253397658.file.myqcloud.com/yield08.png)

1. 执行到`result = c.next()`的时候，这个会执行这个生成器函数直到遇见`yield`语句，yield语句返回值111，控制权里面回到caller()
   里来，将返回值赋值给result，那么打印"(1) result=111"。
2. 然后执行`c.send('Hello')`，这一步会跳到myfunc()的上次保存点，m接收值`Hello`（上一次是从`m = yield 111`
   那个地方跳出来的），然后打印"get m = Hello"，直到运行到又碰到yield语句`yield 222`。
3. 然后控制权返回到caller()来，并将result设置成222，打印"(2) result=222"。
4. 最后一句`c.send('Goodbye')`，又跳到myfunc()中的上次保存点去，n接收值'Goodbye'，然后打印`get n = Goodbye`
   ，它会继续往下执行一定要找到一个`yield`语句并返回到caller()中，否则抛出`StopIteration`异常。

### Python中的协程

现在我们已经对生成器和`yield`原理有了理解。让我们再看看`yield`还有什么是可以做的。
尽管send确实可以像上面那样用，但是生产简单的序列的时候我们基本不会那样用。下面再举一个例子看看send使用的常见场景：

```python
import random


def get_data():
    """Return 3 random integers between 0 and 9"""
    return random.sample(range(10), 3)


def consume():
    """Displays a running average across lists of integers sent to it"""
    running_sum = 0
    data_items_seen = 0

    while True:
        data = yield  # 负责接收数据
        data_items_seen += len(data)
        running_sum += sum(data)
        print('The running average is {}'.format(running_sum / float(data_items_seen)))


def produce(consumer):
    """Produces a set of values and forwards them to the pre-defined consumer
    function"""
    while True:
        data = get_data()
        print('Produced {}'.format(data))
        consumer.send(data)
        yield  # 变成一个生成器函数


if __name__ == '__main__':
    consumer = consume()
    consumer.send(None)
    producer = produce(consumer)

    for _ in range(10):
        print('Producing...')
        next(producer)
```

呃，竟然是生产者-消费者模式，这种方式其实就是协程异步方案，看上去好高级哦，不过原理很简单，应该看得懂吧。

Python的线程是系统级进程，线程间切换开销不小，而且Python在执行多线程时默认加了一个全局解释器锁（GIL），所以Python的多线程其实是串行的，并不能利用多核的优势。而哪怕在Java、C#这样的语言中，多线程真的是并发的，虽然可以利用多核优势，但由于线程的切换是由调度器控制的，不论是用户级线程还是系统级线程，调度器都会由于IO操作、时间片用完等原因强制夺取某个线程的控制权，又由于线程间共享状态的不可控性，同时也会带来安全问题。所以我们在写多线程程序的时候都会加各种锁，很是麻烦，一不小心就会造成死锁，而且锁对性能，总是有些影响的。

然后说到协程，与线程的抢占式调度不同，它是协作式调度，之前忘了看哪个技术分享视频说过，纤程（跟协程一样，只是叫法不同），大致上就是一种可控制的回调，是比线程更轻量级的一种实现异步编程的方式。协程在Python中可以用generator(
生成器)实现，生成器主要由yeild关键字实现。

而用yeild实现协程的话，其实有挺多不同的方式，不过大致的思想都是用yeild来暂停当前执行的程序，转而执行另一个，再在恰当的时候（可以控制）回来执行。用法很灵活。

produce和consumer协同合作，过程清晰可控，在一个线程中执行，不需要锁，也就不会出现死锁情况。

其实Python用生成器对协程的实现并不完整，不过可以用一些第三方库，比如greenlet，毕竟Python最大的优势就是库多嘛。

### 总结

看完这篇文章，我希望你能记住下面这几点就足够了：

* 生成器是用来生产序列的
* `yield`就相对于是生成器函数的`return`
* `yield`额外做的工作就是保存了生成器函数的状态
* 一个生成器是一个特殊的迭代器
* 和迭代器类似，我们可以通过在生成器上面使用`next()`或者是`for`循环来获取下一个值

我希望这篇文章能对你理解生成器有帮助，你应该知道了它是什么，为什么它这样有用，以及怎样使用它。如果你之前就熟悉它，我希望现在对于它不会存在误解了。

