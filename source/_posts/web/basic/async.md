---
title: 使用Ajax实现异步任务
date: 2017-05-02 09:07:13 +0800
toc: true
categories: [ web ]
tags: [ web ]
---

我们经常会遇见许多要运行很长时间的任务，如果还是按照常规的页面请求方式，就会产生卡顿，页面假死现象。
这时候我们第一个想到的就是将这种同步请求方式转换成异步请求，然而对于不需要关心返回结果的请求这个非常简单，
大部分情况是我们还得知道异步任务返回结果，然后调用回调函数来更新页面结果。

目前常见的三种方式是Ajax轮训、Ajax长连接（long polling）、WebSocket方式。
这里我只讲Ajax的两种方式，因为更好的WebSocket方式我已经单独写了一篇文章来介绍。
<!-- more -->

通过一个实际例子来演示下。

## 服务器实现

这里通过一个SpringMVC项目，实现服务器长时间任务部分：

```java
@RequestMapping("/longtime")
    public void ajax(HttpServletRequest request, HttpServletResponse response, long timed) throws Exception {
        PrintWriter writer = response.getWriter();
        Random rand = new Random();
        // 死循环 查询有无数据变化
        while (true) {
            Thread.sleep(300); // 休眠300毫秒，模拟处理业务等
            int i = rand.nextInt(100); // 产生一个0-100之间的随机数
            if (i > 20 && i < 56) { // 如果随机数在20-56之间就视为有效数据，模拟数据发生变化
                long responseTime = System.currentTimeMillis();
                // 返回数据信息，请求时间、返回数据时间、耗时
                writer.print("result: " + i + ", response time: " + responseTime + ", request time: "
                        + timed + ", use time: " + (responseTime - timed));
                break; // 跳出循环，返回数据
            } else { // 模拟没有数据变化，将休眠 hold住连接
                Thread.sleep(1300);
            }
        }
    }
```

服务器端在进行长连接的程序设计时，要注意以下几点：

1. 服务器程序对轮询的可控性。由于轮询是用死循环的方式实现的，所以在算法上要保证程序对何时退出循环有完全的控制能力，
   避免进入死循环而耗尽服务器资源。
2. 合理选择"心跳"频率。长连接必须由客户端不停地进行请求来维持，所以在客户端和服务器间保持正常的"心跳"至为关键，
   参数POLLING_LIFE应小于WEB服务器的超时时间，一般建议在10～20秒左右。也就是ajax请求中timeout: 5000参数
3. 网络因素的影响。在实际应用时，从服务器做出应答，到下一次循环的建立，是有时间延迟的，延迟时间的长短受网络传输等多种因素影响，
   在这段时间内，长连接处于暂时断开的空档，如果恰好有数据在这段时间内发生变动，服务器是无法立即进行推送的，
   所以，在算法设计上要注意解决由于延迟可能造成的数据丢失问题。
4. 服务器的性能。在长连接应用中，服务器与每个客户端实例都保持一个持久的连接，这将消耗大量服务器资源，
   特别是在一些大型应用系统中更是如此，大量并发的长连接有可能导致新的请求被阻塞甚至系统崩溃，
   所以，在进行程序设计时应特别注意算法的优化和改进，必要时还需要考虑服务器的负载均衡和集群技术。

## 轮训方式

### setTimeout和setInterval

它们的定义：

1. setTimeout 指定延迟后调用函数
2. setInterval 以指定周期调用函数

对于`setInterval`而言，如果第一个函数的执行时间特别长，在执行的过程中本应触发了许多个func怎么办，
那么所有这些应该触发的函数都会进入队列吗？

不，只要发现队列中有一个被执行的函数存在，那么其他的统统忽略。如下图，在第300毫秒和400毫秒处的回调都被抛弃，
一旦第一个函数执行完后，接着执行队列中的第二个，即使这个函数已经"过时"很久了。

还有一点，虽然你在setInterval的里指定的周期是100毫秒，但它并不能保证两个函数之间调用的间隔一定是一百毫秒。
在上面的情况中，如果队列中的第二个函数时在第450毫秒处结束的话，在第500毫秒时，它会继续执行下一轮func，
也就是说这之间的间隔只有50毫秒，而非周期100毫秒

那如果我想保证每次执行的间隔应该怎么办？用setTimeout，比如下面的代码：

```js
var i = 1
var timer = setTimeout(function() {
    alert(i++)
    timer = setTimeout(arguments.callee, 2000)
}, 2000)
```

上面的函数每2秒钟递归调用自己一次，你可以在某一次alert的时候等待任意长的时间（不按"确定"按钮），
接下来无论你什么时候点击"确定"， 下一次执行一定离这次确定相差2秒钟的

两者的清除：

```js
t1 = setTimeout(xx(), 1000);
clearTimeout(t1);

t2 = setInterval(yy(), 1000)
clearInteval(t2);
```

### 使用setInterval轮训

客户端实现的就是用一种普通轮询的结果，比较简单。利用`setInterval`不间断的刷新来获取服务器的资源，
这种方式的优点就是简单、及时。

缺点是链接多数是无效重复的；响应的结果没有顺序（因为是异步请求，当发送的请求没有返回结果的时候，后面的请求又被发送。
而此时如果后面的请求比前面的请求要先返回结果，那么当前面的请求返回结果数据时已经是过时无效的数据了）；
请求多，难于维护、浪费服务器和网络资源。

看看实际的轮训例子：

```js
var count1 = 0;
function ask1() {
    var repeat = window.setInterval(function () {
        $.get("${ctx}/longtime.html",
            {"timed": new Date().getTime()},
            function (data) {
                $("#logs").append("[data: " + data + " ]<br/>");
                count1++;
                if (count1 >= 10) {
                    $("#logs").append("Ajax 轮训方式 - finished!");
                    window.clearInterval(repeat);
                }
            });
    }, 3000);
}
```

运行效果：

![](https://xnstatic-1253397658.file.myqcloud.com/ajax01.png)

可以发现，后面请求不会等待前面请求完，也就是也许会同时出现多个HTTP连接服务器，造成服务器资源大量浪费。

## 长连接

这种是Ajax最常见的请求方式，我们通常使用ajax异步提交请求时候就是用的这种方式：

```js
// Ajax long polling 方式
var count2 = 0;
function ask2() {
    $.ajax({
        url: "${ctx}/longtime.html",
        data: {"timed": new Date().getTime()},
        dataType: "text",
        timeout: 5000,
        error: function (XMLHttpRequest, textStatus, errorThrown) {
            $("#state").append("[state: " + textStatus + ", error: " + errorThrown + "]<br/>");
            if (textStatus == "timeout") { // 请求超时
                ask2(); // 递归调用
            } else {
                // 其他错误，如网络错误等
                ask2();
            }
        },
        success: function (data, textStatus) {
            $("#state").append("[state: " + textStatus + ", data:{" + data + "}]<br/>");
            count2++;
            if (count2 >= 10) {
                $("#state").append("Ajax long polling 方式 - finished!");
                return;
            }
            if (textStatus == "success") { // 请求成功
                ask2();
            }
        }
    });
}
```

主要优点就是和服务器始终保持一个连接。如果当前连接请求成功后，将更新数据并且继续创建一个新的连接和服务器保持联系。
如果连接超时或发生异常，这个时候程序也会创建一个新连接继续请求。这样就大大节省了服务器和网络资源，提高了程序的性能，
从而也保证了程序的顺序。

运行效果图：

![](https://xnstatic-1253397658.file.myqcloud.com/ajax02.png)

可以发现，要么前面的运行成功，那么超时，每个时候最多有一个HTTP连接。


