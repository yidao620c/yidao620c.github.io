---
title: SpringBoot系列 - 异步线程池
date: 2017-07-20 12:56:22 +0800
toc: true
categories: [ java ]
tags: [ spring ]
---

在项目中，当访问其他人的接口较慢或者做耗时任务时，不想程序一直卡在耗时任务上，想程序能够并行执行，
我们可以使用多线程来并行的处理任务，也可以使用spring提供的异步处理方式@Async。

Spring异步线程池的接口类，其实质是`java.util.concurrent.Executor`。

Spring 已经实现的异常线程池：

1. `SimpleAsyncTaskExecutor`：不是真的线程池，这个类不重用线程，每次调用都会创建一个新的线程。
2. `SyncTaskExecutor`：这个类没有实现异步调用，只是一个同步操作，只适用于不需要多线程的地方
3. `ConcurrentTaskExecutor`：Executor的适配类，不推荐使用。如果`ThreadPoolTaskExecutor`不满足要求时，才用考虑使用这个类
4. `SimpleThreadPoolTaskExecutor`：是Quartz的`SimpleThreadPool`的类。线程池同时被quartz和非quartz使用，才需要使用此类
5. `ThreadPoolTaskExecutor`：最常使用，推荐。其实质是对`java.util.concurrent.ThreadPoolExecutor`的包装

在异步处理的方法上添加注解`@Async`，就会启动一个新的线程去执行。
<!-- more -->

## 开启异步配置

SpringBoot中开启异步支持非常简单，只需要在配置类上面加上注解`@EnableAsync`，同时定义自己的线程池即可。
也可以不定义自己的线程池，则使用系统默认的线程池。这个注解可以放在Application启动类上，但是更推荐放在配置类上面。

```java
@Configuration
@EnableAsync
public class AsyncConfig implements AsyncConfigurer {
    // 省略...
}
```

异步处理方法分为不返回结果和返回结果，这两者的处理是有区别的。

## 返回void

没有结果返回的示例：

```java
@Component
public class AsyncTask {
    @Async
    public void dealNoReturnTask(){
        log.info("返回值为void的异步调用开始" + Thread.currentThread().getName());
        try {
            Thread.sleep(3000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        log.info("返回值为void的异步调用结束" + Thread.currentThread().getName());
    }
}
```

## 返回Future

异步调用返回数据，Future表示在未来某个点获取执行结果，返回数据类型可以自定义

```java
@Async
public Future<String> dealHaveReturnTask(int i) {
    log.info("asyncInvokeReturnFuture, parementer=" + i);
    Future<String> future;
    try {
        Thread.sleep(1000 * i);
        future = new AsyncResult<String>("success:" + i);
    } catch (InterruptedException e) {
        future = new AsyncResult<String>("error");
    }
    return future;
}
```

以上的异步方法和普通的方法调用相同：

```java
Future<String> future = asyncDemo.asyncInvokeReturnFuture(100);
System.out.println(future.get());
```

## 异常处理

我们可以实现`AsyncConfigurer`接口，也可以继承`AsyncConfigurerSupport`类来实现。

在方法getAsyncExecutor()中创建线程池的时候，必须使用 `executor.initialize()`，不然在调用时会报线程池未初始化的异常。

```java
@Configuration
@EnableAsync
public class AsyncConfig implements AsyncConfigurer {

    @Override
    public Executor getAsyncExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(10);
        executor.setMaxPoolSize(100);
        executor.setQueueCapacity(100);
        executor.setWaitForTasksToCompleteOnShutdown(true);
        executor.setAwaitTerminationSeconds(60 * 10);
        executor.setThreadNamePrefix("AsyncThread-");
        executor.initialize(); //如果不初始化，导致找到不到执行器
        return executor;
    }
    @Override
    public AsyncUncaughtExceptionHandler getAsyncUncaughtExceptionHandler() {
        return new AsyncExceptionHandler();
    }
}
```

异步异常处理类：

```java
public class AsyncExceptionHandler implements AsyncUncaughtExceptionHandler {
    private static final Logger log = LoggerFactory.getLogger(AsyncExceptionHandler.class);
    
    @Override
    public void handleUncaughtException(Throwable ex, Method method, Object... params) {
        log.info("Async method has uncaught exception, params: " + params);

        if (ex instanceof AsyncException) {
            AsyncException asyncException = (AsyncException) ex;
            log.info("asyncException:"  + asyncException.getMsg());
        }

        log.error("Exception :", ex);
    }
}
```

异步处理异常类：

```java
public class AsyncException extends Exception {
    private int code;
    private String msg;
    
    public int getCode() {
        return code;
    }

    public void setCode(int code) {
        this.code = code;
    }

    public String getMsg() {
        return msg;
    }

    public void setMsg(String msg) {
        this.msg = msg;
    }
}
```

在调用方法时，可能出现方法中抛出异常的情况。Spring对于2种异步方法的异常处理机制如下：

1. 对于方法返回值是Futrue的异步方法: a) 在调用future的get时捕获异常; b) 在异常方法中直接捕获异常
2. 对于返回值是void的异步方法：通过`AsyncUncaughtExceptionHandler`处理异常

## 测试代码

最后写个测试代码看看是否跟预期一致：

```java
/**
 * 测试异步任务
 */
@RunWith(SpringRunner.class)
@SpringBootTest
public class ApplicationTests {
    private static final Logger log = LoggerFactory.getLogger(ApplicationTests.class);
    @Autowired
    private AsyncTask asyncTask;

    @Test
    public void testAsync() throws InterruptedException, ExecutionException {
        asyncTask.dealNoReturnTask();
        Future<String> f = asyncTask.dealHaveReturnTask(5);
        log.info("主线程执行finished");
        log.info(f.get());
    }
}
```

执行日志如下：

```
INFO 4180 --- [           main] com.xncoding.pos.ApplicationTests         : 主线程执行finished
INFO 4180 --- [  AsyncThread-2] com.xncoding.pos.async.AsyncTask          : asyncInvokeReturnFuture, parementer=5
INFO 4180 --- [  AsyncThread-1] com.xncoding.pos.async.AsyncTask          : 返回值为void的异步调用开始AsyncThread-1
INFO 4180 --- [  AsyncThread-1] com.xncoding.pos.async.AsyncTask          : 返回值为void的异步调用结束AsyncThread-1
INFO 4180 --- [           main] com.xncoding.pos.ApplicationTests         : success:5
INFO 4180 --- [       Thread-4] o.s.w.c.s.GenericWebApplicationContext   : Closing ....
INFO 4180 --- [       Thread-4] com.alibaba.druid.pool.DruidDataSource   : {dataSource-1} closed
```

根据日志的线程名称很清楚的看出，每个异步方法在线程池的不同线程中执行。

## FAQ

实际运行中，还出现过一个问题，一个Service中的方法调用自己的另一个方法，然后我将这个方法加上@Async注解，然而并不起作用。
异步方法都应该放到单独的异步任务Bean里面去，然后将这个Bean注入到Service中即可。

```java
@Service
public class DeviceService {

    @Resource
    private AsyncTask asyncTask;
    
    public int unbind(Integer id, ManagerInfo managerInfo) {
        // 前面省略...
        
        // 开始异步推送消息
        asyncTask.pushUnbindMsg(managerInfo, pos, location);
    }
}
```

## GitHub源码

[springboot-async](https://github.com/yidao620c/SpringBootBucket/tree/master/springboot-async)

