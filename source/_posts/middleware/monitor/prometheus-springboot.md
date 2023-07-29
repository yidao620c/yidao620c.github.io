---
title: Prometheus监控SpringBoot
date: 2021-02-27 19:15:22 +0800
toc: true
categories: [ 中间件 ]
tags: [ prometheus ]
---

Micrometer 为 Java 平台上的性能数据收集提供了一个通用的 API，
它提供了多种度量指标类型（Timers、Guauges、Counters等），同时支持接入不同的监控系统，
例如 Influxdb、Graphite、Prometheus 等。我们可以通过 Micrometer 收集 Java 性能数据，
配合 Prometheus 监控系统实时获取数据，并最终在 Grafana 上展示出来，从而很容易实现应用的监控。

<span style="color: red;">简单地说，actuator 是真正去采集数据的模块，而 Micrometer 更像是一个适配器，
将 actuator 采集到的数据适合给各种监控工具。</span>

Micrometer 中有两个最核心的概念，分别是计量器（Meter）和计量器注册表（MeterRegistry）。

计量器Meter用来收集不同类型的性能指标信息，Micrometer 提供了如下几种不同类型的计量器：

<!-- more -->

### Counter
计数器，它允许你增加固定的数量，且数量必须为正数，也就是说它描述一个递增的值。也是最常用的一种计量器，
例如接口请求总数、请求错误总数、队列数量变化等。
``` java
// 构造器创建
Counter counter = Counter
    .builder("http.request")
    .baseUnit("num") // optional
    .description("a description of what this counter does") // optional
    .tags("uri", "/order/create") // optional
    .register(registry);
counter.increment();
    
// 直接从Registry创建
MeterRegistry meterRegistry = new SimpleMeterRegistry();
Counter counter = meterRegistry.counter("http.request", "uri", "/order/create");
counter.increment();
```

### Function Counter
特化类型的计数器，Counter的值由某个对象执行某个方法提供，而无需手动调用`counter.increment`。
通常来说主要是用于包装已经存在的计数器或者统计对象，方法需要是单调递增的。
``` java
// 构造器创建
public class FunctionCounterMain {
    public static void main(String[] args) throws Exception {
        MeterRegistry registry = new SimpleMeterRegistry();
        AtomicInteger n = new AtomicInteger(0);
        //这里ToDoubleFunction匿名实现其实可以使用Lambda表达式简化为AtomicInteger::get
        FunctionCounter.builder("functionCounter", n, new ToDoubleFunction<AtomicInteger>() {
            @Override
            public double applyAsDouble(AtomicInteger value) {
                return value.get();
            }
        }).baseUnit("function")
                .description("functionCounter")
                .tag("createOrder", "CHANNEL-A")
                .register(registry);
        //下面模拟三次计数        
        n.incrementAndGet();
        n.incrementAndGet();
        n.incrementAndGet();
    }
}

// 直接从Registry创建
Cache cache = ...; // suppose we have a Guava cache with stats recording on
registry.more().counter("evictions", tags, cache, c -> c.stats().evictionCount());
```
FunctionCounter使用的一个明显的好处是，我们不需要感知FunctionCounter实例的存在，
实际上我们只需要操作作为FunctionCounter实例构建元素之一的AtomicInteger实例即可，
这种接口的设计方式在很多框架里面都可以看到。

### Gauge
Gauge可以理解为直接的数值指标，典型的例子是线程池的活跃线程数量、集合的大小等，
当指标不是递增的而是一个上下浮动的值时，你应该采用Gauge，同时Gauge也翻译为仪表盘，
典型如汽车的速度仪表，这样就非常好理解了。
``` java
// 构造器创建
Gauge gauge = Gauge
    .builder("gauge", myObj, myObj::gaugeValue)
    .description("a description of what this gauge does") // optional
    .tags("region", "test") // optional
    .register(registry);

// 直接从registry创建
registry.gauge("listGauge", Collections.emptyList(), new ArrayList<>(), List::size);
```

### Time Gauge
与Gauge功能相似，但是记录的内容是时间，本质上就是比普通Gauge多了一个时间单位属性。
``` java
// 构造器创建
TimeGauge timeGauge = TimeGauge.builder("timeGauge", count,
        TimeUnit.SECONDS, AtomicInteger::get)
        .tag("tagkey", "tagVal")
        .register(registry);

// 直接从registry创建
registry.more().timeGauge("timeGauge", count, TimeUnit.SECONDS, AtomicInteger::get);
```

### Timer
Timer(计时器)适用于记录耗时比较短的事件的执行时间，通过时间分布展示事件的序列和发生频率。
所有的Timer的实现至少记录了发生的事件的数量和这些事件的总耗时，从而生成一个时间序列。

Timer的基本单位基于服务端的指标而定，但是实际上我们不需要过于关注Timer的基本单位，
因为Micrometer在存储生成的时间序列的时候会自动选择适当的基本单位。

根据个人经验和实践，总结如下：

* 记录指定方法的执行时间用于展示。
* 记录一些任务的执行时间，从而确定某些数据来源的速率，例如消息队列消息的消费速率等。

在实际生产环境中，可以通过spring-aop把记录方法耗时的逻辑抽象到一个切面中，这样就能减少不必要的冗余的模板代码。

``` java
// 构造器创建
Timer timer = Timer
    .builder("my.timer")
    .description("a description of what this timer does") // optional
    .tags("region", "test") // optional
    .register(registry);

// 直接从Registry创建
registry.timer("my.timer", "region", "test");

// record directly
timer.record(Duration.of(60L, ChronoUnit.SECONDS)); 
// record function
timer.record(() -> {
     // some operation ...
}); 
// wrap function
timer.wrap(() -> yourFunction());

// use sample
Timer.Sample sample = Timer.start();
// do something
sample.stop(timer);
```

另外，Timer的使用还可以基于它的内部类Timer.Sample，通过start和stop两个方法记录两者之间的逻辑的执行耗时。例如：
``` java
Timer.Sample sample = Timer.start(registry);
// 这里做业务逻辑
Response response = ...
sample.stop(registry.timer("my.timer", "response", response.status()));
```

### Function Timer
Timer的特化类型，Function Timer由两个单调递增的函数组成，一个用于计数，一个用于统计总耗时。
同样常用于包装已经存在的监控对象。Function Timer在Timer计数功能基础之上增加了每个记录的耗时。
``` java
// 构造器创建
FunctionTimer.builder("cache.gets.latency", cache,
        c -> c.getLocalMapStats().getGetOperationCount(),
        c -> c.getLocalMapStats().getTotalGetLatency(),
        TimeUnit.NANOSECONDS)
    .tags("name", cache.getName())
    .description("Cache gets")
    .register(registry);

// 直接从Registry创建
registry.more().timer("cache.gets.latency", cache,
        c -> c.getLocalMapStats().getGetOperationCount(),
        c -> c.getLocalMapStats().getTotalGetLatency(),
        TimeUnit.NANOSECONDS);
```

### Long Task Timer
Long Task Timer同样也是Timer的特殊类型。统计的是当前有多少正在执行的任务，以及这些这任务已经耗费了多少时间，
适用于监控长时间执行的任务和方法，统计类似当前负载量的相关指标。Long Task Timer在事件开始时记录，
在事件结束后将事件移除。
``` java
// 构造器创建
LongTaskTimer longTaskTimer = LongTaskTimer
    .builder("long.task.timer")
    .description("a description of what this timer does") // optional
    .tags("region", "test") // optional
    .register(registry);

// 直接从Registry创建
LongTaskTimer scrapeTimer = registry.more().longTaskTimer("scrape");

// record function
scrapeTimer.record(() -> {
    // some operation ...
});

// record by sample
LongTaskTimer.Sample start = scrapeTimer.start();
// do something
start.stop();
```

LongTaskTimer适合用于长时间持续运行的事件耗时的记录，例如相对耗时的定时任务。在Spring应用中，
可以简单地使用@Scheduled和@Timed注解，基于spring-aop完成定时调度任务的总耗时记录：
``` java
@Timed(value = "aws.scrape", longTask = true)
@Scheduled(fixedDelay = 360000)
void scrapeResources() {
    //这里做相对耗时的业务逻辑
}
```

当然，在非spring体系中也能方便地使用LongTaskTimer：
``` java
public class LongTaskTimerMain {
    public static void main(String[] args) throws Exception{
        MeterRegistry meterRegistry = new SimpleMeterRegistry();
        LongTaskTimer longTaskTimer = meterRegistry.more().longTaskTimer("longTaskTimer");
        longTaskTimer.record(() -> {
            //这里编写Task的逻辑
        });
        //或者这样
        Metrics.more().longTaskTimer("longTaskTimer").record(()-> {
            //这里编写Task的逻辑
        });
    }
}
```

### Distribution Summary
Distribution Summary翻译为分布概要，主要用于跟踪事件的分布，它的记录形式与Timer十分相似，
但是记录的内容不依赖于时间单位，可以是任意数值，
比如在监测范围内各个Http请求的响应内容大小时就可以使用Distribution Summary。
为了更加明确地表明记录的内容，通常创建Distribution Summary时应该设置baseUnit属性。

分布概要根据每个事件所对应的值，把事件分配到对应的桶（bucket）中。Micrometer 默认的桶的值从 1 到最大的 long 值。
可以通过 minimumExpectedValue 和 maximumExpectedValue 来控制值的范围。如果事件所对应的值较小，
可以通过 scale 来设置一个值来对数值进行放大。与分布概要密切相关的是直方图和百分比（percentile）。
大多数时候，我们并不关注具体的数值，而是数值的分布区间。比如在查看 HTTP 服务响应时间的性能指标时，
通常关注是的几个重要的百分比，如 50%，75%和 90%等。所关注的是对于这些百分比数量的请求都在多少时间内完成。
Micrometer 提供了两种不同的方式来处理百分比。

* 对于 Prometheus 这样本身提供了对百分比支持的监控系统，Micrometer 直接发送收集的直方图数据，
由监控系统完成计算。
* 对于其他不支持百分比的系统，Micrometer 会进行计算，并把百分比结果发送到监控系统。
``` java
public void summary() {
    DistributionSummary summary = DistributionSummary.builder("simple")
        .description("simple distribution summary")
        .minimumExpectedValue(1L)
        .maximumExpectedValue(10L)
        .publishPercentiles(0.5, 0.75, 0.9)
        .register(registry);
    summary.record(1);
    summary.record(1.3);
    summary.record(2.4);
    summary.record(3.5);
    summary.record(4.1);
    System.out.println(summary.takeSnapshot());
}
```

使用场景：不依赖于时间单位的记录值的测量，例如服务器有效负载值，缓存的命中率等。

``` java
// 构造器创建
DistributionSummary summary = DistributionSummary
    .builder("response.size")
    .description("a description of what this summary does") // optional
    .baseUnit("bytes") // optional (1)
    .tags("region", "test") // optional
    .scale(100) // optional (2)
    .register(registry);

// 直接从registry创建
DistributionSummary summary = registry.summary("response.size");
```

## Spring Boot 工程集成 Micrometer
我们一般说 Spring Boot 集成 Micrometer 值得时 Spring 2.x 版本，
因为在该版本 spring-boot-actuator 使用了 Micrometer 来实现监控。

pom.xml配置如下：
``` xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.6.2</version>
        <relativePath/> <!-- lookup parent from repository -->
    </parent>
    <groupId>com.xncoding</groupId>
    <artifactId>springmvc-demo</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>springmvc-demo</name>
    <description>springmvc-demo</description>
    <properties>
        <java.version>11</java.version>
    </properties>
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>
        <dependency>
            <groupId>io.micrometer</groupId>
            <artifactId>micrometer-registry-prometheus</artifactId>
            <scope>runtime</scope>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>
    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
</project>
```

这里引入了 io.micrometer 的 micrometer-registry-prometheus 依赖以及 spring-boot-starter-actuator 依赖，
因为该包对 Prometheus 进行了封装，可以很方便的集成到 Spring Boot 工程中。

其次在 application.properties 中配置如下：
```properties
server.port=8080
spring.application.name=springboot2-prometheus

management.endpoints.web.exposure.include=*
management.endpoint.health.show-details=always
management.endpoint.metrics.enabled=true
management.endpoint.prometheus.enabled=true
management.metrics.export.prometheus.enabled=true
management.metrics.tags.application=${spring.application.name}
```

这里 `management.endpoints.web.exposure.include=*` 配置为开启 Actuator 服务，
因为Spring Boot Actuator 会自动配置一个 URL 为 `/actuator/prometheus` 的 HTTP 服务来供 
Prometheus 抓取数据，不过默认该服务是关闭的，该配置将打开所有的 Actuator 服务。
`management.metrics.tags.application` 配置会将该工程应用名称添加到计量器注册表的 tag 中去，
方便后边 Prometheus 根据应用名称来区分不同的服务。

然后在工程启动主类中添加 Bean 如下来监控 JVM 性能指标信息：
``` java
@SpringBootApplication
public class SpringmvcDemoApplication {
    public static void main(String[] args) {
        SpringApplication.run(SpringmvcDemoApplication.class, args);
    }
    @Bean
    MeterRegistryCustomizer<MeterRegistry> configurer(@Value("${spring.application.name}") String applicationName) {
        return registry -> registry.config().commonTags("application", applicationName);
    }
}
```

最后，启动服务，浏览器访问 `http://127.0.0.1:8080/actuator/prometheus`
就可以看到应用的一系列不同类型 metrics 信息，例如 `http_server_requests_seconds summary`、
`jvm_memory_used_bytes gauge`、`jvm_gc_memory_promoted_bytes_total counter` 等等。

> [!TIP]
> 程序中配置的Meter名称转换后会改变，我们都可以通过访问`/actuator/prometheus`这个链接来查看所有的Query类型。

## 配置 Prometheus 监控应用指标
修改 prometheus.yml 配置，在上篇文章配置示例基础上，添加上边启动的服务地址来执行监控。重启Prometheus容器。
```yaml
global:
  scrape_interval: 10s

scrape_configs:
  - job_name: node
    static_configs:
      - targets: ['service:9100']
  - job_name: 'application'
    scrape_interval: 5s
    metrics_path: '/actuator/prometheus'
    static_configs:
      - targets: ['192.168.31.160:8080']
```

## 配置 Grafana Dashboard 展示监控项
Prometheus 现在已经可以正常监控到应用 JVM 信息了，那么我们可以配置 Grafana Dashboard 来优雅直观的展示出来这些监控值了。
需要导入对应的监控 JVM 的 Dashboard 模板，模板编号为 4701。数据源选择上面配置好的Prometheus即可。
![img.png](https://xnstatic-1253397658.file.myqcloud.com/20230715-13.png)

## 自定义监控指标并展示到 Grafana
上边是 spring-boot-actuator 集成了 Micrometer 来提供的默认监控项，覆盖 JVM 各个层间的监控，
配合 Grafana Dashboard 模板基本可以满足我们日常对 Java 应用的监控。当然，它也支持自定义监控指标，
实现各个方面的监控，例如统计访问某一个 API 接口的请求数，统计实时在线人数、统计实时接口响应时间等功能，
而这些都可以通过使用上边的四种计量器来实现。接下来，来演示下如何自定义监控指标并展示到 Grafana 上。

### 监控某几个 API 请求次数
我们继续在springboot2工程上添加 IndexController.java，来实现分别统计访问 index 及 core 接口请求次数。
``` java
@RestController
@RequestMapping("/v1")
public class IndexController {

    private final MeterRegistry registry;

    private Counter counter_core;
    private Counter counter_index;

    public IndexController(MeterRegistry registry) {
        this.registry = registry;
    }

    @PostConstruct
    private void init() {
        counter_core = registry.counter("app_requests_method_count", "method", "IndexController.core");
        counter_index = registry.counter("app_requests_method_count", "method", "IndexController.index");
    }

    @RequestMapping(value = "/index")
    public Object index() {
        try {
            counter_index.increment();
        } catch (Exception e) {
            return e;
        }
        return counter_index.count() + " index of springboot2-prometheus.";
    }

    @RequestMapping(value = "/core")
    public Object coreUrl() {
        try {
            counter_core.increment();
        } catch (Exception e) {
            return e;
        }
        return counter_core.count() + " coreUrl Monitor by Prometheus.";
    }
}
```

说明一下，这里是一个简单的 RestController 接口，使用了 Counter 计量器来统计访问 /v1/index 及 
/v1/core 接口访问量。因为访问数会持续的增加，所以这里使用 Counter 比较合适。启动服务，
我们来分别访问一下这两个接口，为了更好的配合下边演示，可以多访问几次。

接下来，我们可以到 Prometheus UI 界面上使用 PromSQL 查询自定义的监控信息了。
分别添加 Graph 并执行如下查询语句，查询结果如下：
![img.png](https://xnstatic-1253397658.file.myqcloud.com/20230715-14.png)

* app_requests_method_count_total 为上边代码中设置的 Counter 名称。
* application 为初始化 registry 时设置的通用标签，标注应用名称，这样做好处就是可以根据应用名称区分不同的应用。
* method 为上边代码中设置的 Counter 标签名称，可以用来区分不同的方法，这样就不用为每一个方法设置一个 Counter 了。

接下来，我们在 Grafana Dashboard 上添加一个新的 Panel 并添加 Query 查询，最后图形化展示出来。

首先添加一个Row并命名为`自定义监控指标`。然后添加一个 Panel ，点击 Add Query 增加一个新的 Query 查询，
查询语句为上边的 PromSQL 语句，不过这里为了更好的扩展性，我们可以将 application 及 instance 两个参数赋值为变量，
而这些变量可以直接从 Prometheus 上传递过来，最终的查询语句为
```
app_requests_method_count_total{application="$application", instance="$instance", method="IndexController.core"}
```
最后修改Panel的 Title 为 `实时访问量 /v1/core`，保存一下，返回首页就可以看到刚添加的 Dashboard 了，是不是很直观。
![img.png](https://xnstatic-1253397658.file.myqcloud.com/20230715-15.png)

### 监控所有 API 请求次数
上边针对某个或某几个接口请求次数做了监控，如果我们想针对整个应用监控所有接口请求总次数，这个该如何实现呢？
监控请求次数可以继续使用 Counter 计数器，整个应用所有请求，我们自然而然的想到了 Spring AOP，
通过切面注入可以做到统计所有请求记录，添加依赖如下：
``` xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-aop</artifactId>
</dependency>
```

添加一个切面：
``` java
@Component
@Aspect
public class AspectAop {
    @Autowired
    MeterRegistry registry;

    private Counter counter_total;

    ThreadLocal<Long> startTime = new ThreadLocal<>();

    @Pointcut("execution(public * com.xncoding.springmvcdemo.controller.*.*(..))")
    private void pointCut() {
    }

    @PostConstruct
    public void init() {
        counter_total = registry.counter("app_requests_count", "v1", "core");
    }

    @Before("pointCut()")
    public void doBefore(JoinPoint joinPoint) {
        startTime.set(System.currentTimeMillis());
        counter_total.increment();
    }

    @AfterReturning(returning = "returnVal", pointcut = "pointCut()")
    public void doAftereReturning(Object returnVal) {
        System.out.println("请求执行时间：" + (System.currentTimeMillis() - startTime.get()));
    }
}
```
在之前的Row中添加一个新的Panel，Title设置为`实时所有请求总量`，添加Query为如下：
```
app_requests_count_total{application="$application", instance="$instance", v1="core"}
```
同时修改图形的样式。如下：
![img.png](https://xnstatic-1253397658.file.myqcloud.com/20230715-16.png)

### 监控实时在线人数
接下来，来演示下如何监控瞬时数据变化，例如实时交易总金额，实时网络请求响应时间，实时在线人数等，
这里我们简单模拟一下实时在线人数监控，这里采用 Gauge 计量仪来做为指标统计类型，
在 IndexController.java 中添加相关代码如下：
``` java
private AtomicInteger app_online_count;

@PostConstruct
private void init() {
    app_online_count = registry.gauge("app_online_count", new AtomicInteger(0));
}

@RequestMapping(value = "/online")
public Object onlineCount() {
    int people = 0;
    try {
        // 通过一个随机数来模拟在线人数
        people = new Random().nextInt(2000);
        app_online_count.set(people);
    } catch (Exception e) {
        return e;
    }
    return "current online people: " + people;
}
```

重启服务，访问一下 `/v1/online` 接口，得到一个 2000 以内的随机数作为实时在线人数。

我们在 Prometheus UI 界面执行一下 PromeSQL 查询语句，同样能够对应获取到实时数据。
```
app_online_count{application="springboot2-prometheus"}
```

继续在 Grafana 上之前的Row里增加一个新的Panel，并选择 Query 语句为上面的查询语句。将图表类型选择为仪表盘类型。
![img.png](https://xnstatic-1253397658.file.myqcloud.com/20230715-17.png)

