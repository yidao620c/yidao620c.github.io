---
title: JVM工具-可视化工具VisualVM
date: 2021-04-25 14:12:55 +0800
toc: true
categories: [ java ]
tags: [ jvm ]
---

VisualVM是功能最强大的运行监控和故障处理工具之一，曾经很长一段时间是Oracle官方主力发展的虚拟机故障处理工具。
它除了常规的运行监视、故障处理外，还提供性能分析（Profiling）。VisualVM不需要被监视程序基于特殊Agent去运行，
因此它的通用性极强，对应用程序实际性能影响很小，使得它能直接应用在生产环境中。

从JDK9开始，不再打包到JDK中，需要单独下载。

<!-- more -->

## 使用方法
VisualVM基于NetBeans平台开发，具备通过插件扩展功能的能力，可以做到

1. 显示虚拟机进程以及进程的配置、环境信息。
2. 监视应用程序的处理器、垃圾收集、堆、方法区以及线程的信息。
3. dump以及分析堆转储快照能力。
4. 方法级程序性能分析，找出被调用最多、运行时间最长的方法。
5. 离线程序快照：收集程序的运行时配置、线程dump、内存dump等信息建立一个快照，将快照导出。
6. 其他扩展插件带来的无限可能性。

官方下载地址：<https://visualvm.github.io/download.html>

下载后，解压后。修改下`/etc/visualvm.conf`文件，将`visualvm_jdkhome=`修改成本机JDK目录地址即可。
双击bin目录下的visualvm.exe即可运行。

初次启动时并未加载任何插件，所以第一步肯定是安装插件。安装插件可以手动也可以自动安装。
推荐在网络环境下，通过点击工具 > 插件菜单，弹出插件页签，在里面的可用插件以及已安装中都有。

选择一个需要监视的程序进入程序主界面。
![img.png](https://xnstatic-1253397658.file.myqcloud.com/img-2021100513.png)

### 生成、浏览堆转储快照
在应用程序窗口双击某个进程后，点击Monitor页签，然后点击Heap Dump即可生成dump文件。
生成后会在该堆的应用程序下增加一个以[heap-dump]开头的子节点，并在主页签中打开该转储快照。
如果想要把dump文件导出，则点击另存为即可。
![img.png](https://xnstatic-1253397658.file.myqcloud.com/img-2021100514.png)

在Summary标签页看到应用程序dump的运行时参数、System.getProperties()内容、线程堆栈等信息。
切换至Objects标签可查看所有对象的信息。OQL是对象查询语句，需要下载对应插件。Threads标签查看线程信息。

### 分析程序性能
在Profiler页签中，VisualVM提供了程序运行期方法级的处理器执行时间分析以及内存分析。
一般不会在生产环境使用这项功能，或者改用JMC来完成。

要开始性能分析，选择CPU和内存按钮中的一个，然后切换到应用程序中对程序进行操作，
VisualVM会记录这段时间内应用程序执行过的所有方法。如果是进行处理器执行时间分析，
将会统计每个方法执行次数、执行耗时；如果是内存分析，则会统计每个方法关联的对象数以及占用空间。
监控结束后，点击停止按钮即可。
![img.png](https://xnstatic-1253397658.file.myqcloud.com/img-2021100515.png)

### BTrace动态日志跟踪
BTrace是一个很神奇的VisualVM插件，它本身也是一个可运行的独立程序。
其作用是在不中断目标程序运行前提下，通过HotSpot虚拟机的Instrument功能动态增加原本不存在的调试代码。
这项功能在实际生产环境中非常有用，当生产环境中程序出现问题，排错时又忘记打印日志了。
这时候不需要在源码中添加日志，然后发包，再替换、重启服务。可在运行期间就加入日志代码解决问题。

安装完BTrace插件后，在应用查询面板中右键点击要调试的应用查询，会出现Trace Application的菜单。
点击后进入BTrace面板。这个面板看上去就是一个简单额Java程序开发环境，里面甚至还有一小段代码。
![img.png](https://xnstatic-1253397658.file.myqcloud.com/img-2021100516.png)

比如我的演示程序的API接口中会调用如下的代码
``` java
private void sleep(int time) {
    try {
        System.out.println("start to do task");
        Thread.sleep(time);
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
}
```
现在我想知道传入的参数time的值是什么。下面是TracingScript脚本代码
``` java
/* BTrace Script Template */
import org.openjdk.btrace.core.annotations.*;
import static org.openjdk.btrace.core.BTraceUtils.*;

@BTrace
public class TracingScript {
    @OnMethod(clazz="com.xncoding.pos.controller.UserController",
    method="sleep",
    location=@Location(Kind.RETURN))
    public static void func(@Self com.xncoding.pos.controller.UserController instance, int time) {
        println("调用堆栈");
        // jstack();
        println(strcat("方法入参time=", str(time)));
        // 强烈建议加上，否则会因为输出缓冲看不到最新打印结果
        println("==========================");
    }
}
```

点击Start按钮后稍等片刻，出现Done表示成功。当程序运行时，这里output就会打印调试信息。

![img.png](https://xnstatic-1253397658.file.myqcloud.com/img-2021100517.png)

BTrace用途很广泛，打印调用堆栈、参数、返回值只是最基本的，在它网站上有性能监视、定位连接泄露、内存泄露、
解决多线程竞争问题等使用案例，可去它的官网了解。

BTrace能实现动态修改程序行为，是因为它基于Java虚拟机的Instrument开发的。Instrument是JVM工具接口的重要组件。
提供了一套代理机制，是的第三方工具程序可以代理方式访问和修改Java虚拟机内部的数据，
阿里巴巴可以的诊断工具Arthas也是通过Instrument实现了跟BTrace类似的功能。

## OQL的语法支持
Visual VM提供了对OQL（对象查询语言）的支持，以便于开发人员在庞大的堆内存数据中，快速定位所需的资源。

官方使用文档：<http://cr.openjdk.java.net/~sundar/8022483/webrev.01/raw_files/new/src/share/classes/com/sun/tools/hat/resources/oqlhelp.html>

### 基本查询语法
OQL 语言是一种类似SQL的查询语言。基本语法如下：
```
select <JavaScript expression to select>
[ from [instanceof] <class name> <identifier>
[ where <JavaScript boolean expression to filter> ] ]
```

OQL由3个部分组成：select 子句、from 子句和where 子句。select 子句指定查询结果要显示的内容；from 子句指定查询范围，
可指定类名，如`java.lang.String`、`char[]`、`[Ljava.io.File`（File数组）；where 子句用于指定查询条件。

select 子句和where 子句支持使用Javascript 语法处理较为复杂的查询逻辑；
select 子句可以使用类似json的语法输出多个列；from子句中可以使用instanceof关键字，将给定类的子类也包括到输出列表中。

在Visual VM的OQL中，可以直接访问对象的属性和部分方法。如下例中，筛选出长度大于等于100的字符串：
```
select s from java.lang.String s where s.value.length > 100
```

选取长度大于等于256的 int 数组：
```
select a from [I a where a.length >= 256
```

![img.png](https://xnstatic-1253397658.file.myqcloud.com/img-2021100701.png)

筛选出表示两位数整数的字符串：
```
select {instance: s, content: s.toString()} from java.lang.String s where /^\d{2}$/.test(s.toString())
```
上例中，select 子句使用了json语法，指定输出两列为String对象以及String.toString() 的输出。
where 子句使用正则表达式，指定了符合`/^\d{2}$/`条件的字符串。

查询文件对象的path路径
```
select file.path.value.toString() from java.io.File file
```

下例使用 instance 关键字选取所有的ClassLoader，包括子类：
```
select classof(cl).name from instanceof java.lang.ClassLoader cl
```

由于在Java程序中一个类可能会被多个ClassLoader同时载入，因此这种情况下，可能需要使用Class的ID来指定Class。
如下例，选出了所有ID为0x37A014D8的Class对象实例。
```
select o from instanceof 0xd404b198 o
```

### 内置heap对象查询
heap对象是 Visual VM OQL 的内置对象。通过 heap 对象可以实现一些强大的OQL功能。heap 对象的主要方法如下：

* forEachClass()：对每一个Class对象执行一个回调操作。它的使用方法类似于 heap.forEachClass(callback)，其中 callback 为 Javascript 函数。
* findClass()：查找给定名称的类对象，返回类的方法和属性参考下表，它的调用方法类似 heap.findClass(className)。
* classes()：返回堆快照中所有的类集合。使用方法如 heap.classes()。
* objects()：返回堆快照中所有的对象集合。使用方法如 heap.objects(clazz,[includeSubtypes],[filter])，
其中clazz指定类名称，includeSubtypes指定是否选出子类，filter 为过滤器，指定筛选规则。includeSubtypes 和 filter 可以省略。
* livepaths()：返回指定对象的存活路径。即，显示哪些对象直接或者间接引用了给定对象。它的使用方法如heap.livepaths(obj)。
* roots()：返回这个堆的根对象。使用方法如heap.roots()。

使用findClass()返回的Class对象拥有的属性和方法 ：

属性                             | 方法
-----------------------------|---------------------------------------
name：类名称                     | isSubclassOf()：是否是指定类的子类
superclass：父类                 | isSuperclassOf()：是否是指定类的父类
statics：类的静态变量的名称和值     | subclasses()：返回所有子类
fields：类的域信息                 | superclasses()：返回所有父类

下例查找java.util.Vector类：
```
select heap.findClass("java.util.Vector")
```

查找java.util.Vector的所有父类：
```
select heap.findClass("java.util.Vector").superclasses()
```

查找所有在java.io包下的类：
```
select filter(heap.classes(), "/java.io./.test(it.name)")
```

查找字符串“56”的引用链：
```
select heap.livepaths(s) from java.lang.String s where s.toString()=='56'
```

如下是一种可能的输出结果，其中java.lang.String#1600即字符串“56”。它显示了该字符串被一个WebPage对象持有。
```
java.lang.String#1600->package.WebPage#57->java.lang.Object[]#341->java.util.Vector#11->package.Student#3
```

下例查找当前堆中所有java.io.File对象实例，参数true表示java.io.File的子类也需要被显示：
```
select heap.objects("java.io.File",true)
```

下例访问了TraceStudent类的静态成员webpages对象：
```
select heap.findClass("package.TraceStudent").webpagess
```

查找系统属性
```
select heap.findClass("java.lang.System").statics.props
```

### 对象函数
在Visual VM中，为OQL语言还提供了一组以对象为操作目标的内置函数。通过这些函数，可以获取目标对象的更多信息。本节主要介绍一些常用的对象函数。

**classof()函数**

返回给定Java对象的类。调用方法形如classof(objname)。返回的类对象有以下属性。

* name：类名称。
* superclass：父类。
* statics：类的静态变量的名称和值。
* fields：类的域信息。

同时返回的Class对象又有如下的方法可以继续调用

* isSubclassOf()：是否是指定类的子类。
* isSuperclassOf()：是否是指定类的父类。
* subclasses()：返回所有子类。
* superclasses()：返回所有父类。

下例将返回所有Vector类以及子类的类型：
```
select classof(v) from instanceof java.util.Vector v
```

**objectid()函数**

objectid()函数返回对象的ID。使用方法如objectid(objname)。

返回所有Vector对象（不包含子类）的ID：
```
select objectid(v) from java.util.Vector v
```

**reachables()函数**

eachables()函数返回给定对象的可达对象集合。使用方法如reachables(obj,[exclude])。obj为给定对象，
exclude指定忽略给定对象中的某一字段的可达引用。

下例返回'56'这个String对象的所有可达对象：
```
select reachables(s) from java.lang.String s where s.toString()=='56'
select reachables(s, "java.lang.String.value") from java.lang.String s where s.toString()=='56'
```

**referrers()函数**

返回引用给定对象的对象集合。使用方法如：referrers(obj)。

下例返回了引用“56”String对象的对象集合：
```
select referrers(s) from java.lang.String s where s.toString()=='56'
```

**referees()函数**

referees()函数返回给定对象的直接引用对象集合，用法形如：referees(obj)。

例返回了File对象的静态成员引用：
```
select referees(heap.findClass("java.io.File"))
```

**sizeof()函数**

sizeof()函数返回指定对象的大小（不包括它的引用对象），即浅堆（Shallow Size）。

注意：sizeof()函数返回对象的大小不包括对象的引用对象。因此，sizeof()的返回值由对象的类型决定，
和对象的具体内容无关。

下例返回所有Vector的大小以及对象：
```
select {size:sizeof(o),Object:o} from java.util.Vector o
```

可以看到，不论Vector集合包含多少对象。Vector对象所占用的内存大小始终为36字节。这是由Vector本身的结构决定的，
与其内容无关。sizeof()函数就是返回对象的固有大小。

**rsizeof()函数**

rsizeof()函数返回对象以及其引用对象的大小总和，即深堆（Retained Size）。这个数值不仅与类本身的结构有关，
还与对象的当前数据内容有关。

下例显示了所有Vector对象的Shallow Size以及Retained Size：

```
select {size:sizeof(o),rsize:rsizeof(o)} from java.util.Vector o
```

输出结果
```
{
size = 36,
rsize = 140
}

{
size = 36,
rsize = 140
}

{
size = 36,
rsize = 140
}
```

注意：resizeof()取得对象以及其引用对象的大小总和。因此，它的返回值与对象的当前数据内容有关。

**toHtml()函数**

toHtml()函数将对象转为HTML显示。

下例将Vector对象的输出使用HTML进行加粗和斜体显示：
```
select "<b><em>"+toHtml(o)+"</em></b>" from java.util.Vector o
```

### 集合/统计函数

Visual VM中还有一组用于集合操作和统计的函数。可以方便地对结果集进行后处理或者统计操作。
集合/统计函数主要有contains()、count()、filter()、length()、map()、max()、min()、sort()、top()等。

下例返回被 File 对象引用的 String 对象集合。首先通过 referrers(s) 得到所有引用String 对象的对象集合。
使用 contains() 函数及其参数布尔等式表达式classof(it).name == 'java.io.File')，
将 contains() 的筛选条件设置为类名是java.io.File 的对象。

```
select s.toString() from java.lang.String s where contains(referrers(s), "classof(it).name == 'java.io.File'")
```

通过该OQL，得到了当前堆中所有的File对象的文件名称。可以理解为当前Java程序通过java.io.File获得已打开或持有的所有文件。

**count()函数**

count()函数返回指定集合内满足给定布尔表达式的对象数量。它的基本使用方法如：count(set, [boolexpression])。
参数set指定要统计总数的集合，boolexpression为布尔条件表达式，可以省略，但如果指定，
count()函数只计算满足表达式的对象个数。在boolexpression表达式中，可以使用以下内置对象。

* it：当前访问对象。
* index：当前对象索引。
* array：当前迭代的数组/集合。

下例返回堆中所有java.io包中的类的数量，布尔表达式使用正则表达式表示。
```
select count(heap.classes(), "/java.io./.test(it.name)")
```

下列返回堆中所有类的数量。
```
select count(heap.classes())
```

filter()函数返回给定集合中，满足某一个布尔表达式的对象子集合。使用方法形如filter(set, boolexpression)。
在boolexpression中，可以使用以下内置对象。

* it：当前访问对象。
* index：当前对象索引。
* array：当前迭代的数组/集合。

下例返回所有java.io包中的类。
```
select filter(heap.classes(), "/java.io./.test(it.name)")
```

下例返回了当前堆中，引用了java.io.File对象并且不在java.io包中的所有对象实例。
首先使用referrers()函数得到所有引用java.io.File对象的实例，接着使用filter()函数进行过滤，
只选取不在java.io包中的对象。
```
select filter(referrers(f), "! /java.io./.test(classof(it).name)") from java.io.File f
```

map()函数将结果集中的每一个元素按照特定的规则进行转换，以方便输出显示。
使用方法形如：map(set, transferCode)。set为目标集合，transferCode为转换代码。
在transferCode中可以使用以下内置对象。

* it：当前访问对象。
* index：当前对象索引。
* array：当前迭代的数组/集合。

下例将当前堆中的所有File对象进行格式化输出：
```
select map(heap.objects("java.io.File"), "index + '=' + it.path.toString()")
```

top()函数返回在给定集合中，按照特定顺序排序的前几个对象。一般使用方法为：top(set, expression,num)。
其中set为给定集合，expression为排序逻辑，num指定输出前几个对象。在expression中，可以使用以下内置对象。

* lhs：用于比较的左侧元素。
* rhs：用于比较的右侧元素。

下例显示了长度最长的前5个字符串：
```
select top(filter(heap.objects('java.lang.String'),'it.value!=null'), 'rhs.value.length - lhs.value.length', 5)
```

sum()函数用于计算集合的累计值。它的一般使用方法为：sum(set,[expression])。其中第一个参数set为给定集合，
参数expression用于将当前对象映射到一个整数，以便用于求和。参数expression可以省略，
如果省略，则可以使用map()函数作为替代。

下例计算所有 java.util.Properties 对象的可达对象的总大小：
```
select sum(map(reachables(p), 'sizeof(it)')) from java.util.Properties p
```

unique()函数将除去指定集合中的重复元素，返回不包含重复元素的集合。它的一般使用方法形如unique(set)。

下例返回当前堆中，有多个不同的字符串：
```
select count(unique(map(heap.objects('java.lang.String'), 'it.value')))
```

### 程序化OQL
Visual VM不仅支持在OQL控制台上执行OQL查询语言，也可以通过其OQL相关的JAR包，将OQL查询程序化，
从而获得更加灵活的对象查询功能，实现堆快照分析的自动化。

在进行OQL开发前，工程需要引用Visual VM安装目录下JAR包。

这里以分析Tomcat堆溢出文件为例，展示程序化OQL带来的便利。 对于给定的Tomcat堆溢出Dump文件，
这里将展示如何通过程序，计算Tomcat平均每秒产生的session数量，代码如下：

``` java
public class AveLoadTomcatOOM {
    public static final String dumpFilePath = "d:/tmp/tomcat_oom/tomcat.hprof";

    public static void main(String args[]) throws Exception {
        OQLEngine engine;
        final List<Long> creationTimes = new ArrayList<Long>(000);
        engine = new OQLEngine(HeapFactory.createHeap(new File(dumpFilePath)));
        String query = "select s.creationTime from org.apache.catalina. session.StandardSession s"; //第8行
        engine.executeQuery(query, new OQLEngine.ObjectVisitor() {
            public boolean visit(Object obj) {
                creationTimes.add((Long) obj);
                return false;
            }
        });
        Collections.sort(creationTimes);
        long min = creationTimes.get(0) / 1000;//第18行
        long max = creationTimes.get(creationTimes.size() - 1) / 1000;
        System.out.println("平均压力：" + creationTimes.size() * 1.0 / (max - min) + "次/秒");
    }
}
```

上述代码第8行，通过OQL语句得到所有session的创建时间，
在第18、19行获得所有session中最早创建和最晚创建的session时间，在第21行计算整个时间段内的平均session创建速度。

使用这种方式可以做到堆转存文件的全自动化分析，并将结果导出到给定文件，当有多个堆转存文件需要分析时，有着重要的作用。

除了使用以上方式外，Visual VM的OQL控制台也支持直接使用JavaScript代码进行编程，如下代码实现了相同功能：
```
var sessions=toArray(heap.objects("org.apache.catalina.session.StandardSession"));
var count=sessions.length;
var createtimes=new Array();
for(var i=0;i<count;i++){
    createtimes[i]=sessions[i].creationTime;
}
createtimes.sort();
var min=createtimes[0]/1000;
var max=createtimes[count-1]/1000;
count/(max-min)+"次/秒"
```
