---
title: redis笔记03 - 进阶篇
date: 2015-07-12 16:15:42 +0800
toc: true
categories: [中间件]
tags: [ redis ]
---

redis中的事务transaction是一组命令集合，要么都执行，要么都不执行。

```
# MULTI
# SADD "user:1:following" 2
# SADD "user:2:followers" 1
# EXEC
```

redis将客户端发送的事务执行链放入一个队列queue中，然后接受到EXEC请求后才顺序执行这个命令串，同时保证执行这些的时候不被其他命令打扰。

错误处理：

* 语法错误，redis直接返回错误，连语法正确的其他命令也不会执行
* 运行错误，只有那条出错的语句不会执行成功，其他照样执行

redis木有rollback机制，这个要靠自己去处理出错情况。
<!-- more -->

### watch命令介绍

我们知道在一个事务中只有当所有的命令都执行完后才能得到每个结果的返回值，可有些情况下需要先获得一条命令的返回值，然后再根据这个值执行下一条命令。这个时候可以使用watch，watch命令可以监控一个或多个键，一旦其中有一个键被修改或删除，之后的事务就不会执行。监控一直持续到EXEC命令(
因为事务中的命令是在EXEC之后才执行的，所以在MULTI命令后可以修改watch监控的键值)

Tips：watch命令的作用只是当被监控的键值被修改后阻止之后一个事务的执行，而不能保证其他客户端不修改这一键值，所以我们需要在EXEC执行失败后重新执行整个函数

执行EXEC命令后会取消对所有键的监控，如果不想执行事务中的命令也可以使用unwatch命令取消监控

### 生存时间

在实际开发中经常会遇到一些有时效的数据，比如限时优惠活动、缓存或验证码等，过了一定时间就需要删除这些数据。在redis中可以使用EXPIRE命令设置一个键的生存时间，到时间后redis自动删除它。

```
# EXPIRE key seconds
# TTL key
```

返回一个键还要多久就要超时，当键不存在或者木有为键设置超时间接时返回-1

```
# PERSIST key
```

取消键的生存时间设置，即将键恢复成永久

除了PERSIST外，使用SET或GETSET命令为键赋值会同时清除键的生存时间。其他命令不会

```
# PEXPIRE key millseconds
```

以毫秒为单位的超时设置

```
# EXPIREAT key time
```

使用Unix时间作为参数，也就是1970-01-01 00:00:00 到现在的秒数

### 实现缓存cache

为了提高网站的负载能力，常常需要将一些访问频率较高但是对CPU或IO资源消耗较大的操作结果缓存起来，比如每次获取一个单方报价，需要去数据库检索一个大字段，请求IO资源，然后多线程反序列化为一个对象，请求线程资源和CUP资源，那么这个东西就需要缓存起来。
一般来讲会给缓存设置过期时间，防止内存被大量占用。

然而在一些场景中这种方法并不能满足需要，当服务器内存有限的时候，如果大量使用缓存键其生存时间过长会导致redis占满内存，另一方面如果生存时间过短会导致缓存命中率过低而影响效率。

为此，可以限制redis能够使用的最大内存，并让redis按照一定的规则淘汰不需要的缓存键，这种方式在只将redis用做缓存系统时候非常实用，哈哈哈哈，原来用做缓存只是redis的一个小小部分功能而已。

具体方法如下：

修改配置文件的maxmemory参数，单位是字节，当超过这个限制redis会根据maxmemory-policy参数指定的策略删除不需要的键，直到redis占用内存小于指定内存。

redis支持的淘汰键规则：

```
* volatile-lru    使用LRU算法删除一个键（只对设置了生存时间的键）
* allkeys-lru    使用LRU算法删除一个键，这个比较常用
* volatile-random    随机删除一个键（只对设置了生存时间的键）
* allkeys-random    随机删除一个键
* volatile-ttl    删除生存时间最近的一个键（只对设置了生存时间的键）
* noeviction    不删除键，只返回错误
```

事实上redis并不会准确将整个数据库中最久未被使用的键删除，而是每次从数据库中随机取3个键并删除这3个键中最久未被使用的键，删除生存时间最接近的键的实现方案也是一样，这个3这个数字可以通过maxmemory-samples来配置。

### 排序

#### 有序集合SortedSet

SortedSet常见使用场景是大数据排序，如游戏玩家排行榜，所以很少会需要获得键中全部数据。可以使用下列命令序列来获取集合运算：

```
MULTI
ZINTERSCORE tempKey ...
ZRANGE tempKey ...
DEL tempKey
EXEC
```

#### SORT命令

sort命令可以对list、Set、SortedSet进行排序，并且可以完成类似于关系型数据库中的连接查询任务。

```
# SORT key ALPHA DESC LIMIT 11, 10
```

默认SORT以数字排序，添加ALPHA参数可以以字典顺序排序，DESC表示倒序，LIMIT参数不解释

#### BY参数

很多情况下list或set中存储的元素值代表的是对象的ID，更多时候我们希望根据ID对应的对象的某个属性进行排序，比如提交时间。
如果提供了BY参数，SORT命令将不再依据元素自身的值进行排序，而是对每个元素使用元素的值替换参考键中第一个*并获取其值，然后依据该值对元素排序。

```
# SORT tag:Python:posts by post:*->time DESC
```

上面的post:*是散列键Hash类型，而time是散列键值中的一个属性

#### GET参数

对于SORT排序后获取了一个ID列表，要显示所有文章的标题，并不需要对每个ID都去请求一次HGET，可以使用SORT命令的GET参数，可以获取多个字段

```
# SORT tag:python:posts BY post:*->time DESC GET post:*->title GET post:*->time GET #
```

最后一个# 代表还能获取文章的ID，也就是返回元素本身的值

#### STORE参数

```
# SORT tag:python:posts BY post:*->time DESC GET post:*->title GET post:*->time GET # STORE sort.result
```

STORE参数常用来结合EXPIRE命令缓存排序结果

#### SORT性能优化：

SORT是redis中最强大最复杂的命令之一，如果使用不好很容易成为性能瓶颈。SORT命令的时间复杂度是O(n + mlogm)
，其中n表示要排序的列表中的元素个数，m表示要返回的元素个数。当n较大时候SORT命令性能会比较低，并且redis在排序前会建立一个长度为n的容器来存储待排序元素，n较大时严重影响性能
所以要注意：

* 尽可能减少待排序键中元素数量
* 使用LIMIT参数只获取需要的数据，m尽量小，分页查询
* 如果要排序的数据量较大，尽可能使用STORE参数将结果缓存起来。

### 消息通知

一个很常见的例子就是发送邮件。使用redis的list数据结构可以轻松实现，消费者LPUSH到list中，然后消费者RPOP取出任务。
一般来讲消费者需要去不断的轮训这个队列，这个有些不完美。如果有新任务加入到任务队列就通知消费者就好了。其实借助BRPOP命令可以实现这样的需求，不过这个是阻塞的，一直到这个队列中有任务了才会返回。

```
# BRPOP queue, timetoutseconds
```

### 优先级队列

BRPOP命令可以同时接受多个键，其完整命令格式为 BRPOP key1 key2

可同时检测多个键，如果所有键都没有元素则阻塞，如果其中有一个键有元素则会从该键中弹出元素。

借此特性可以实现区分优先级的任务队列。

```
$task = BRPOP queue:confirm.email queue:notify.email 0
execute($task[1])
```

这样只要confirm队列中有任务，那么取出来的肯定是confirm任务，否则才会去取notify的任务

### 发布/订阅模式

```
# PUBLISH channel message
向某个频道channel发布一个消息
# SUBSCRIBE channel
# UNSUBSCRIBE channel
按照规则订阅：
支持glob风格的通配符，shell风格的通配符，跟正则式没有半毛钱关系
# PSUBSCRIBE channel.?*
```

### 管道

客户端和redis使用TCP协议连接。往返时延在数量级上相当于redis处理一条简单命令。
redis底层通信协议对管道pipelining提供了支持，但一组命令中每条命令都不依赖于之前命令的执行结果就可以将这组命令一起通过管道发出去。

### 内存优化

内部编码优化
