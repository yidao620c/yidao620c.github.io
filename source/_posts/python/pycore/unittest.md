---
title: python核心 - 单元测试
date: 2015-12-22 22:22:22 +0800
toc: true
categories: [ python ]
tags: [ python核心 ]
---

单元测试在所有编程语言中都不陌生，对于一个健壮的软件来讲单元测试是很有必要的，
并且"测试驱动开发"（TDD：Test-Driven Development）越来越受欢迎也说明了它的重要性。

单元测试一个最大的好处，就是确保一个程序模块的行为符合我们设计的测试用例。
在将来修改的时候，可以极大程度地保证该模块行为仍然是正确的。
<!-- more -->

## 基本用法

我们通过一个例子来说明怎样写单元测试，先编写一个Dict类，它的行为和dic一致，但是可以通过属性来访问。

```python
class Dict(dict):

    def __init__(self, **kw):
        super().__init__(**kw)

    def __getattr__(self, key):
        try:
            return self[key]
        except KeyError:
            raise AttributeError(r"'Dict' object has no attribute '%s'" % key)

    def __setattr__(self, key, value):
        self[key] = value
```

python自带unittest模块来方便我们编写单元测试，先引入：

```python
import unittest


class TestDict(unittest.TestCase):

    def test_init(self):
        d = Dict(a=1, b='test')
        self.assertEqual(d.a, 1)
        self.assertEqual(d.b, 'test')
        self.assertTrue(isinstance(d, dict))

    def test_key(self):
        d = Dict()
        d['key'] = 'value'
        self.assertEqual(d.key, 'value')

    def test_attr(self):
        d = Dict()
        d.key = 'value'
        self.assertTrue('key' in d)
        self.assertEqual(d['key'], 'value')

    def test_keyerror(self):
        d = Dict()
        with self.assertRaises(KeyError):
            value = d['empty']

    def test_attrerror(self):
        d = Dict()
        with self.assertRaises(AttributeError):
            value = d.empty
```

编写单元测试时，我们需要编写一个测试类，从unittest.TestCase继承。
以test开头的方法就是测试方法，不以test开头的方法不被认为是测试方法，测试的时候不会被执行。

对每一类测试都需要编写一个test_xxx()方法。由于unittest.TestCase提供了很多内置的条件判断，
我们只需要调用这些方法就可以断言输出是否是我们所期望的。最常用的断言就是assertEqual()：

```python
# 断言函数返回的结果与1相等
self.assertEqual(abs(-1), 1)
```

另一种重要的断言就是期待抛出指定类型的Error，比如通过d['empty']访问不存在的key时，断言会抛出KeyError：

```python
with self.assertRaises(KeyError):
    value = d['empty']
```

## 运行单个测试

一旦编写好单元测试，我们就可以运行单元测试。最简单的运行方式是在mydict_test.py的最后加上两行代码：

```python
if __name__ == '__main__':
    unittest.main()
```

这种方式可将其当做普通的python脚本来运行。

另一种方法是在命令行通过参数-m unittest直接运行单元测试，无需写main方法（推荐这种方式）：

```python
python - m
unittest
mydict_test
```

## 运行测试套件

最常见的情况是软件很多，需要为每个模块写一个单元测试模块，
最后想将它们作为一个测试套件一起运行，那么就要使用TestLoader了：

```python
import unittest

if __name__ == '__main__':
    loader = unittest.TestLoader()
    start_dir = './test'
    suite = loader.discover(start_dir)
    runner = unittest.TextTestRunner()
    runner.run(suite)
```

上面通过使用TestLoader自动发现测试用例并运行。

## setUp与tearDown

可以在单元测试中编写两个特殊的setUp()和tearDown()方法。这两个方法会分别在每调用一个测试方法的前后分别被执行。

setUp()和tearDown()方法有什么用呢？设想你的测试需要启动一个数据库，这时，就可以在setUp()方法中连接数据库，
在tearDown()方法中关闭数据库，这样，不必在每个测试方法中重复相同的代码。

另外，如果你想在整个测试类所有方法开始执行前和所有方法执行完后执行，就使用类方法`setUpClass()`和`tearDownClass()`:

```python
class TestDict(unittest.TestCase):

    def setUp(self):
        print('setUp...')

    def tearDown(self):
        print('tearDown...')

    @classmethod
    def setUpClass(cls):
        cls._connection = createExpensiveConnectionObject()

    @classmethod
    def tearDownClass(cls):
        cls._connection.destroy()

```

如果是模块级别的，就使用函数setUpModule()和tearDownModule()：

```python
def setUpModule():
    createConnection()


def tearDownModule():
    closeConnection()
```

## 忽略测试

你还可以根据某些条件来忽略某些测试，这里使用@unittest.skip注解实现：

```python
class MyTestCase(unittest.TestCase):

    @unittest.skip("demonstrating skipping")
    def test_nothing(self):
        self.fail("shouldn't happen")

    @unittest.skipIf(mylib.__version__ < (1, 3),
                     "not supported in this library version")
    def test_format(self):
        # Tests that work for only a certain version of the library.
        pass

    @unittest.skipUnless(sys.platform.startswith("win"), "requires Windows")
    def test_windows_support(self):
        # windows specific testing code
        pass
```

还可以忽略整个测试类：

```python
@unittest.skip("showing class skipping")
class MySkippedTestCase(unittest.TestCase):
    def test_not_run(self):
        pass
```

标注某个测试一定会失败:

```python
@unittest.expectedFailure


def test_fail(self):
    self.assertEqual(1, 0, "broken")
```

## assert方法列表

`TestCase`提供了很多个断言方法，我们分基本部分来列出来。

首先是最常用的：

 Method                    | Checks that          
---------------------------|----------------------
 assertEqual(a, b)         | a == b               
 assertNotEqual(a, b)      | a != b               
 assertTrue(x)             | bool(x) is True      
 assertFalse(x)            | bool(x) is False     
 assertIs(a, b)            | a is b               
 assertIsNot(a, b)         | a is not b           
 assertIsNone(x)           | x is None            
 assertIsNotNone(x)        | x is not None        
 assertIn(a, b)            | a in b               
 assertNotIn(a, b)         | a not in b           
 assertIsInstance(a, b)    | isinstance(a, b)     
 assertNotIsInstance(a, b) | not isinstance(a, b) 

测试异常：

 Method                                                              | Checks that                                                                      
---------------------------------------------------------------------|----------------------------------------------------------------------------------
 @% raw %@assertRaises(exc, fun, *args, **kwds)@% endraw %@          | @% raw %@fun(*args, **kwds) raises exc@% endraw %@                               
 @% raw %@assertRaisesRegexp(exc, r, fun, *args, **kwds)@% endraw %@ | @% raw %@fun(*args, **kwds) raises exc and the message matches regex@% endraw %@ 

其他高级用法：

 Method                      | Checks that        
-----------------------------|--------------------
 assertAlmostEqual(a, b)     | round(a-b, 7) == 0 
 assertNotAlmostEqual(a, b)  | round(a-b, 7) != 0 
 assertRegexpMatches(s, r)   | r.search(s)        
 ssertNotRegexpMatches(s, r) | not r.search(s)    

## 深入理解unittest

前面讲了这个多单元测试的基本使用方法，而深入理解它的基本原理很重要。
要理解unittest框架，有4个概念需要搞清楚：test fixture, test case, test suite, test runner

下面是静态类图：
![](https://xnstatic-1253397658.file.myqcloud.com/pyunittest01.png)

* 一个TestCase的实例就是一个测试用例。什么是测试用例呢？就是一个完整的测试流程，包括测试前准备环境的搭建(setUp)，
  执行测试代码(run)，以及测试后环境的还原(tearDown)。元测试(unit test)的本质也就在这里，
  一个测试用例是一个完整的测试单元，通过运行这个测试单元，可以对某一个问题进行验证。
  而多个测试用例集合在一起，就是TestSuite，而且TestSuite也可以嵌套TestSuite。
* TestLoader是用来加载TestCase到TestSuite中的，其中有几个loadTestsFrom__()方法，
  就是从各个地方寻找TestCase，创建它们的实例，然后add到TestSuite中，再返回一个TestSuite实例。
* TextTestRunner是来执行测试用例的，其中的run(test)会执行TestSuite/TestCase中的run(result)方法。
* 测试的结果会保存到TextTestResult实例中，包括运行了多少测试用例，成功了多少，失败了多少等信息。

这样整个流程就清楚了，首先是要写好TestCase，然后由TestLoader加载TestCase到TestSuite，
然后由TextTestRunner来运行TestSuite，运行的结果保存在TextTestResult中，整个过程集成在unittest.main模块中。

现在已经涉及到了test case, test suite, test runner这三个概念了，还有test fixture没有提到，那什么是test fixture呢？
在TestCase的docstring中有这样一段话：

```
Test authors should subclass TestCase for their own tests.
Construction and deconstruction of the test's environment ('fixture') can be implemented by
overriding the 'setUp' and 'tearDown' methods respectively.
```

可见，对一个测试用例环境的搭建和销毁，包含测试用例本身就是一个fixture，
通过覆盖TestCase的setUp()和tearDown()方法来实现。

下面这个图可以更加清晰的描述它们之间的关系：
![](https://xnstatic-1253397658.file.myqcloud.com/pyunittest02.png)

下面通过简单的例子再来实践一下，就拿unittest文档上的例子吧：

```python
import random
import unittest


class TestSequenceFunctions(unittest.TestCase):

    def setUp(self):
        self.seq = range(10)

    def test_shuffle(self):
        # make sure the shuffled sequence does not lose any elements
        random.shuffle(self.seq)
        self.seq.sort()
        self.assertEqual(self.seq, range(10))

        # should raise an exception for an immutable sequence
        self.assertRaises(TypeError, random.shuffle, (1, 2, 3))

    def test_choice(self):
        element = random.choice(self.seq)
        self.assertTrue(element in self.seq)

    def test_sample(self):
        with self.assertRaises(ValueError):
            random.sample(self.seq, 20)
        for element in random.sample(self.seq, 5):
            self.assertTrue(element in self.seq)


if __name__ == '__main__':
    unittest.main()
```

这个TestSequenceFunctions类到底是个什么呢？它是一个测试用例，还是三个测试用例？
其实，我们只要看一些TestLoader是如何加载测试用例的，就一清二楚了，
在`loader.TestLoader`类中有一个`loadTestsFromTestCase()`方法：

```python
def loadTestsFromTestCase(self, testCaseClass):
    """Return a suite of all tests cases contained in testCaseClass"""
    if issubclass(testCaseClass, suite.TestSuite):
        raise TypeError("Test cases should not be derived from TestSuite."
                        " Maybe you meant to derive from TestCase?")
    testCaseNames = self.getTestCaseNames(testCaseClass)
    if not testCaseNames and hasattr(testCaseClass, 'runTest'):
        testCaseNames = ['runTest']
    loaded_suite = self.suiteClass(map(testCaseClass, testCaseNames))
    return loaded_suite
```

getTestCaseNames()是从TestCase这个类中找所有以"test"开头的方法，然后注意第9行，在构造TestSuite对象时，
其参数使用了一个map方法，即对testCaseNames中的每一个元素，使用testCaseClass为其构造对象，
其结果是一个TestCase的对象集合。

在loader.TestLoader类会为每个以`test`开头的方法创建一个TestCase对象。
如果没有定义test开头的方法，而是将测试代码写到了一个名为runTest的方法中，
那么会为该runTest方法构建TestCase对象，如果定义了test开头的方法，就会忽略runTest方法。

至此，基本就清楚了，每一个以test开头的方法，都会为其构建TestCase对象，
也就是说TestSequenceFunctions类中其实定义了三个TestCase，之所以写成这样，是为了方便，
因为这几个测试用例的fixture是相同的，如果每一个测试用例单独写成一个TestCase的话，会有很多的冗余代码。

## 生成测试报告

unittest本身并不具备这个功能，需要使用[HTMLTestRunner](https://pypi.python.org/pypi/HTMLTestRunner)库

1. 首先需要下载.py文件：<http://tungwaiyip.info/software/HTMLTestRunner.html>
2. 下载后放入python安装目录的lib文件夹下面。
3. 打开终端进入python交互模式导入HTMLTestRunner,如果无导入错误显示，则说明添加成功

通过`help(HTMLTestRunner)`来显示使用帮助，非常简单，可以修改源码自己定制。

### 报告中显示用例的注释

给报告中的每个测试用例添加注释，来说明该测试用例是用来干什么的，非常有必要。
这里在每个测试函数的下方添加上注释：

```python
def test_equal(self):
    '''这里是测试两个值是否相等'''
    self.assertEqual(1, 1)
```

运行后，打开生成的html文件可以看到，每个测试用例函数的后面有该用例的注释。

