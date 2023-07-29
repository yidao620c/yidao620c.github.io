---
title: SpringBoot系列 - 定时任务
date: 2017-07-12 10:59:25 +0800
toc: true
categories: [ java ]
tags: [ spring ]
---

很多时候，我们需要在每天的某个固定时间或者每隔一段时间让应用去执行某一个任务。
为了实现这个需求，通常我们会通过多线程来实现这个功能，但是这样我们需要自己做一些比较麻烦的工作。
接下来，让我们看看如何使用Spring scheduling task简化定时任务功能的实现。

默认，springboot已经支持了定时任务Schedule模块，所以一般情况已经完全能够满足我们的实际需求，
一般来说，没有必要再加入Quartz2了，不过你要是有更高级需求也可以整合Quartz2进来。
<!-- more -->

## 定时任务架构

一般来说，实际项目中，为了提高服务的响应能力，我们一般会通过负载均衡的方式，或者反向代理多个节点的方式来进行。
如果我们将定时任务写在我们的项目中，那么在同一个时间点，定时任务会一起执行，也就是会执行多次，这样很可能会导致我们的业务出现错误。

建议使用逻辑分离的方式来解决这个问题。就是我们将真正要定时任务处理的逻辑，写成rest或者rpc服务，
然后我们可以新建一个单独的定时任务项目，这个项目应该是没有任何的业务代码的，他纯粹只有定时任务功能，
几点启动，或者每隔多少时间启动。启动后，通过rest或者rpc的方式，调用真正处理逻辑的服务。

## 添加依赖

实际上只要添加最基础的start依赖即可支持定时任务：

```xml

<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>2.0.4.RELEASE</version>
</parent>
```

## 添加注解配置类

如果有多个耗时任务，最好使用线程池来执行，添加一个配置类专门用来配置定时任务执行的线程池：

```java
@Configuration
@EnableScheduling
public class ScheduleConfig implements SchedulingConfigurer {
    @Override
    public void configureTasks(ScheduledTaskRegistrar taskRegistrar) {
        taskRegistrar.setScheduler(taskExecutor());
    }

    @Bean(destroyMethod="shutdown")
    public ExecutorService taskExecutor() {
        return Executors.newScheduledThreadPool(20);
    }
}
```

## 添加执行任务类

```java
@Component
public class MyJob {
    private Logger logger = LoggerFactory.getLogger(MyJob.class);
    public final static long ONE_Minute =  10 * 1000;

    @Scheduled(fixedDelay=ONE_Minute)
    public void fixedDelayJob() throws InterruptedException {
        logger.info(DateTimeKit.formatDateTime(new Date())+" >>fixedDelay执行.... start");
        Thread.sleep(5000L);
    }

    @Scheduled(fixedRate=ONE_Minute)
    public void fixedRateJob() throws InterruptedException {
        logger.info(DateTimeKit.formatDateTime(new Date())+" >>fixedRate执行....");
        Thread.sleep(8000L);
    }

    /**
     * 第一位，表示秒，取值0-59
     * 第二位，表示分，取值0-59
     * 第三位，表示小时，取值0-23
     * 第四位，日期天/日，取值1-31
     * 第五位，日期月份，取值1-12
     * 第六位，星期，取值1-7，1表示星期天，2表示星期一
     * 第七位，年份，可以留空，取值1970-2099
     */
    @Scheduled(cron="0 40 19 * * ?")
    public void cronJob(){
        logger.info(DateTimeKit.formatDateTime(new Date())+" >>cron执行....");
    }
}
```

## 三种定时任务类型

上面的例子中有三种典型的定时任务，先将前面最简单的两种，单位都是毫秒，比如1分钟=60秒×1000：

1. fixedDelay：当任务执行完毕后x毫秒后再执行下一个任务
2. fixedRate： 每隔x毫秒执行一次，不论你业务执行花费了多少时间

而还有一类定时任务，比如是每天的3点15分执行，那么我们就需要用另外一种方式：cron表达式

## cron表达式

cron表达式，有专门的语法，而且感觉有点绕人，不过简单来说，大家记住一些常用的用法即可，特殊的语法可以单独去查。

cron一共有7位，但是最后一位是年，可以留空，所以我们可以写6位：

```
* 第一位，秒，取值0-59
* 第二位，分，取值0-59
* 第三位，时，取值0-23
* 第四位，日，取值1-31
* 第五位，月，取值1-12
* 第六位，星期，取值1-7，1表示星期天，2表示星期一
* 第七位，年份，可以留空，取值1970-2099

```

cron中，还有一些特殊的符号，含义如下：

```
(*)星号：可以理解为每的意思，每秒，每分，每时，每日，每月，每星期，每年...
(?)问号：问号只能出现在日和星期这两个位置，表示这个位置的值不确定，每天3点执行，所以第六位星期的位置，
我们是不需要关注的，就是不确定的值。同时：日和星期是两个相互排斥的元素，通过问号来表明不指定值。
比如，1月10日，比如是星期1，如果在星期的位置是另指定星期二，就前后冲突矛盾了。
(-)减号：表达一个范围，如在小时字段中使用"10-12"，则表示从10到12点，即10,11,12
(,)逗号：表达一个列表值，如在星期字段中使用"1,2,4"，则表示星期一，星期二，星期四
(/)斜杠：如：x/y，x是开始值，y是步长，比如在第一位（秒） 0/15就是，从0秒开始，每15秒，最后就是0，15，30，45，60 另：*/y，等同于0/y
(L)字符：用在日表示一个月中的最后一天，用在周表示该月最后一个星期X
(W)字符：指定离给定日期最近的工作日(周一到周五)
(#)字符：表示该月第几个周X。6#3表示该月第3个周五
```

下面列举几个例子供大家来验证：

```
0 0 3 * * ?     每天3点执行
0 5 3 * * ?     每天3点5分执行
0 5 3 ? * *     每天3点5分执行，与上面作用相同
0 5/10 3 * * ?  每天3点的 5分，15分，25分，35分，45分，55分这几个时间点执行
0 10 3 ? * 1    每周星期天，3点10分 执行，注：1表示星期天    
0 10 3 ? * 1#3  每个月的第三个星期，星期天执行，#号只能出现在星期的位置
```

## GitHub源码

[springboot-schedule](https://github.com/yidao620c/SpringBootBucket/tree/master/springboot-schedule)

