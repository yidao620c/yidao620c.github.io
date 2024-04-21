---
title: SpringBoot系列 - 使用消息队列RabbitMQ
date: 2017-08-06 12:33:21 +0800
toc: true
categories: [ java ]
tags: [ spring ]
---

RabbitMQ 即一个消息队列，主要是用来实现应用程序的异步和解耦，同时也能起到消息缓冲，消息分发的作用。

RabbitMQ是实现AMQP（高级消息队列协议）的消息中间件的一种，AMQP，即Advanced Message Queuing Protocol，
高级消息队列协议，是应用层协议的一个开放标准，为面向消息的中间件设计。

在项目中，将一些无需即时返回且耗时的操作提取出来，进行了异步处理，而这种异步处理的方式大大的节省了服务器的请求响应时间，
提高了系统的吞吐量。

本篇将详细介绍RabbitMQ以及如何在SpringBoot中使用。
<!-- more -->

## 简单概念

关于消息中间件RabbitMQ的详细介绍可以参考我的博客系列教程：[RabbitMQ简易教程](https://www.xncoding.com/tags/rabbitmq/)

这里我只对一些最核心的概念拿出来讲一下，不理解这些就根本无法理解下面的代码。

几个名词术语：

* Broker - 简单来说就是消息队列服务器的实体。
* Exchange - 消息路由器，转发消息到绑定的队列上，指定消息按什么规则，路由到哪个队列。
* Queue - 消息队列，用来存储消息，每个消息都会被投入到一个或多个队列。
* Binding - 绑定，它的作用就是把 Exchange 和 Queue 按照路由规则绑定起来。
* RoutingKey - 路由关键字，Exchange 根据这个关键字进行消息投递。
* Producter - 消息生产者，产生消息的程序。
* Consumer - 消息消费者，接收消息的程序。
* Channel - 消息通道，在客户端的每个连接里可建立多个Channel，每个channel代表一个会话。

一般消息队列都是生产者将消息发送到队列，消费者监听队列进行消费。rabbitmq中一个虚拟主机（默认 /）持有一个或者多个交换机（Exchange）。
用户只能在虚拟主机的粒度进行权限控制，交换机根据一定的策略（RoutingKey）绑定(Binding)到队列(Queue)上，
这样生产者和队列就没有直接联系，而是将消息发送的交换机，交换机再把消息转发到对应绑定的队列上。

上面说了交换机（Exchange）作为rabbitmq的一个独特的重要的概念，单独拿出来讲一下。我们用到的最常见的是4中类型：

1. Direct: 先匹配, 再投送。即在绑定时设定一个routing_key, 消息的routing_key匹配时, 才会被交换器投送到绑定的队列中去.
   交换机跟队列必须是精确的对应关系，这种最为简单。
1. Topic: 转发消息主要是根据通配符。在这种交换机下，队列和交换机的绑定会定义一种路由模式，那么，通配符就要在这种路由模式和路由键之间匹配后交换机才能转发消息
   这种可以认为是Direct 的灵活版
1. Headers: 也是根据规则匹配, 相较于 direct 和 topic 固定地使用 routingkey , headers则是一个自定义匹配规则的类型，
   在队列与交换器绑定时会设定一组键值对规则，消息中也包括一组键值对( headers属性)，当这些键值对有一对或全部匹配时，消息被投送到对应队列
1. Fanout : 消息广播模式，不管路由键或者是路由模式，会把消息发给绑定给它的全部队列，如果配置了routingkey会被忽略

理解交换机如何投送消息的原理后，后面的就比较简单了。

## Ack机制

在RabbitMQ的消息队列编程中，有个非常重要的概率叫消息确认机制，也就是Ack机制，这里我需要单独讲一下。

每个Consumer可能需要一段时间才能处理完收到的数据。如果在这个过程中，Consumer出错了或者异常退出了，而数据还没有处理完成，那么非常不幸，这段数据就丢失了。

因为我们采用no-ack的方式进行确认，也就是说，每次Consumer接到数据后，而不管是否处理完成，RabbitMQ
Server会立即把这个Message标记为完成，然后从queue中删除了。
如果一个Consumer异常退出了，它处理的数据能够被另外的Consumer处理，这样数据在这种情况下就不会丢失了（注意是这种情况下）。

为了保证数据不被丢失，RabbitMQ支持消息确认机制，即acknowledgments。为了保证数据能被正确处理而不仅仅是被Consumer收到，那么我们不能采用no-ack。而应该是在处理完数据后发送ack。
在处理数据后发送的ack，就是告诉RabbitMQ数据已经被接收，处理完成，RabbitMQ可以去安全的删除它了。

如果Consumer退出了但是没有发送ack，那么RabbitMQ就会把这个Message发送到下一个Consumer。这样就保证了在Consumer异常退出的情况下数据也不会丢失。

这里并没有用到超时机制。RabbitMQ仅仅通过Consumer的连接中断来确认该Message并没有被正确处理。也就是说，RabbitMQ给了Consumer足够长的时间来做数据处理。
这样即使你通过Ctr-C中断了Recieve.cs，那么Message也不会丢失了，它会被分发到下一个Consumer。

如果忘记了ack，那么后果很严重。当Consumer退出时，Message会重新分发。然后RabbitMQ会占用越来越多的内存，由于RabbitMQ会长时间运行，因此这个"
内存泄漏"是致命的。

下面的例子中我会使用手动Ack模式。

## 环境准备

RabbitMQ需要安装erlang和RabbitMQ Server，
安装教程请参考：[RabbitMQ简易教程 - 安装](https://www.xncoding.com/2017/05/06/mq/rabbitmq-tutorial01.html)

请先按照教程安装好RabbitMQ后，添加一个用户spring/123456，后面配置要用到。

## SpringBoot中集成

接下来正式开始讲解如何在SpringBoot中集成了。

### maven依赖

第一版当然是先添加maven依赖了，有官方现成的starter依赖，如下：

```xml

<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-amqp</artifactId>
    </dependency>
    <dependency>
        <groupId>com.fasterxml.jackson.core</groupId>
        <artifactId>jackson-databind</artifactId>
        <version>2.9.6</version>
    </dependency>

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <exclusions>
            <exclusion>
                <groupId>com.vaadin.external.google</groupId>
                <artifactId>android-json</artifactId>
            </exclusion>
        </exclusions>
    </dependency>
</dependencies>
```

### 配置文件

在application.yml中添加rabbitmq相关配置：

```yaml
spring:
  rabbitmq:
    host: 127.0.0.1
    port: 5672
    username: spring
    password: 123456
    publisher-confirms: true #支持发布确认
    publisher-returns: true  #支持发布返回
    listener:
      simple:
        acknowledge-mode: manual #采用手动应答
        concurrency: 1 #指定最小的消费者数量
        max-concurrency: 1 #指定最大的消费者数量
        retry:
          enabled: true #是否支持重试
```

### 配置类

最核心的类就是RabbitMQ的配置类，在里面定制模版类、声明交换机、队列、绑定交换机到队列。

这里我声明了一个Direct类型的交换机，并通过路由键绑定到一个队列中来测试Direct模式，
另外还声明了Fanout类型的交换机，并绑定到2个队列来测试广播模式。

```java
@Configuration
public class RabbitConfig {
    @Resource
    private RabbitTemplate rabbitTemplate;

    /**
     * 定制化amqp模版      可根据需要定制多个
     * <p>
     * <p>
     * 此处为模版类定义 Jackson消息转换器
     * ConfirmCallback接口用于实现消息发送到RabbitMQ交换器后接收ack回调   即消息发送到exchange  ack
     * ReturnCallback接口用于实现消息发送到RabbitMQ 交换器，但无相应队列与交换器绑定时的回调  即消息发送不到任何一个队列中  ack
     *
     * @return the amqp template
     */
    // @Primary
    @Bean
    public AmqpTemplate amqpTemplate() {
        Logger log = LoggerFactory.getLogger(RabbitTemplate.class);
        // 使用jackson 消息转换器
        rabbitTemplate.setMessageConverter(new Jackson2JsonMessageConverter());
        rabbitTemplate.setEncoding("UTF-8");
        // 消息发送失败返回到队列中，yml需要配置 publisher-returns: true
        rabbitTemplate.setMandatory(true);
        rabbitTemplate.setReturnCallback((message, replyCode, replyText, exchange, routingKey) -> {
            String correlationId = message.getMessageProperties().getCorrelationIdString();
            log.debug("消息：{} 发送失败, 应答码：{} 原因：{} 交换机: {}  路由键: {}", correlationId, replyCode, replyText, exchange, routingKey);
        });
        // 消息确认，yml需要配置 publisher-confirms: true
        rabbitTemplate.setConfirmCallback((correlationData, ack, cause) -> {
            if (ack) {
                log.debug("消息发送到exchange成功,id: {}", correlationData.getId());
            } else {
                log.debug("消息发送到exchange失败,原因: {}", cause);
            }
        });
        return rabbitTemplate;
    }

    /* ----------------------------------------------------------------------------Direct exchange test--------------------------------------------------------------------------- */

    /**
     * 声明Direct交换机 支持持久化.
     *
     * @return the exchange
     */
    @Bean("directExchange")
    public Exchange directExchange() {
        return ExchangeBuilder.directExchange("DIRECT_EXCHANGE").durable(true).build();
    }

    /**
     * 声明一个队列 支持持久化.
     *
     * @return the queue
     */
    @Bean("directQueue")
    public Queue directQueue() {
        return QueueBuilder.durable("DIRECT_QUEUE").build();
    }

    /**
     * 通过绑定键 将指定队列绑定到一个指定的交换机 .
     *
     * @param queue    the queue
     * @param exchange the exchange
     * @return the binding
     */
    @Bean
    public Binding directBinding(@Qualifier("directQueue") Queue queue,
                                 @Qualifier("directExchange") Exchange exchange) {
        return BindingBuilder.bind(queue).to(exchange).with("DIRECT_ROUTING_KEY").noargs();
    }

    /* ----------------------------------------------------------------------------Fanout exchange test--------------------------------------------------------------------------- */

    /**
     * 声明 fanout 交换机.
     *
     * @return the exchange
     */
    @Bean("fanoutExchange")
    public FanoutExchange fanoutExchange() {
        return (FanoutExchange) ExchangeBuilder.fanoutExchange("FANOUT_EXCHANGE").durable(true).build();
    }

    /**
     * Fanout queue A.
     *
     * @return the queue
     */
    @Bean("fanoutQueueA")
    public Queue fanoutQueueA() {
        return QueueBuilder.durable("FANOUT_QUEUE_A").build();
    }

    /**
     * Fanout queue B .
     *
     * @return the queue
     */
    @Bean("fanoutQueueB")
    public Queue fanoutQueueB() {
        return QueueBuilder.durable("FANOUT_QUEUE_B").build();
    }

    /**
     * 绑定队列A 到Fanout 交换机.
     *
     * @param queue          the queue
     * @param fanoutExchange the fanout exchange
     * @return the binding
     */
    @Bean
    public Binding bindingA(@Qualifier("fanoutQueueA") Queue queue,
                            @Qualifier("fanoutExchange") FanoutExchange fanoutExchange) {
        return BindingBuilder.bind(queue).to(fanoutExchange);
    }

    /**
     * 绑定队列B 到Fanout 交换机.
     *
     * @param queue          the queue
     * @param fanoutExchange the fanout exchange
     * @return the binding
     */
    @Bean
    public Binding bindingB(@Qualifier("fanoutQueueB") Queue queue,
                            @Qualifier("fanoutExchange") FanoutExchange fanoutExchange) {
        return BindingBuilder.bind(queue).to(fanoutExchange);
    }
}
```

### 监听器

接下来编写消息队列的监听器类，监听队列消息并做相应的处理，并通过Ack机制确认处理完成：

```java
/**
 * 消息监听器
 *
 * @author XiongNeng
 * @version 1.0
 */
@Component
public class Receiver {
    private static final Logger log = LoggerFactory.getLogger(Receiver.class);

    /**
     * FANOUT广播队列监听一.
     *
     * @param message the message
     * @param channel the channel
     * @throws IOException the io exception  这里异常需要处理
     */
    @RabbitListener(queues = {"FANOUT_QUEUE_A"})
    public void on(Message message, Channel channel) throws IOException {
        channel.basicAck(message.getMessageProperties().getDeliveryTag(), true);
        log.debug("FANOUT_QUEUE_A " + new String(message.getBody()));
    }

    /**
     * FANOUT广播队列监听二.
     *
     * @param message the message
     * @param channel the channel
     * @throws IOException the io exception   这里异常需要处理
     */
    @RabbitListener(queues = {"FANOUT_QUEUE_B"})
    public void t(Message message, Channel channel) throws IOException {
        channel.basicAck(message.getMessageProperties().getDeliveryTag(), true);
        log.debug("FANOUT_QUEUE_B " + new String(message.getBody()));
    }

    /**
     * DIRECT模式.
     *
     * @param message the message
     * @param channel the channel
     * @throws IOException the io exception  这里异常需要处理
     */
    @RabbitListener(queues = {"DIRECT_QUEUE"})
    public void message(Message message, Channel channel) throws IOException {
        channel.basicAck(message.getMessageProperties().getDeliveryTag(), true);
        log.debug("DIRECT " + new String(message.getBody()));
    }
}
```

### 消息发送者

再写一个消息发送服务，用来向交换机发送消息：

```java
/**
 * 消息发送服务
 */
@Service
public class SenderService {
    private Logger logger = LoggerFactory.getLogger(this.getClass());
    @Resource
    private RabbitTemplate rabbitTemplate;

    /**
     * 测试广播模式.
     *
     * @param p the p
     * @return the response entity
     */
    public void broadcast(String p) {
        CorrelationData correlationData = new CorrelationData(UUID.randomUUID().toString());
        rabbitTemplate.convertAndSend("FANOUT_EXCHANGE", "", p, correlationData);
    }

    /**
     * 测试Direct模式.
     *
     * @param p the p
     * @return the response entity
     */
    public void direct(String p) {
        CorrelationData correlationData = new CorrelationData(UUID.randomUUID().toString());
        rabbitTemplate.convertAndSend("DIRECT_EXCHANGE", "DIRECT_ROUTING_KEY", p, correlationData);
    }

}
```

## 运行测试

代码写完后当然要写个测试用例测一下嘛。

```java
/**
 * SenderServiceTest
 *
 * @author XiongNeng
 * @version 1.0
 */
@RunWith(SpringRunner.class)
@SpringBootTest(classes = Application.class)
public class SenderServiceTest {
    @Autowired
    private SenderService senderService;

    @Test
    public void testCache() {
        // 测试广播模式
        senderService.broadcast("同学们集合啦！");
        // 测试Direct模式
        senderService.direct("定点消息");
    }
}
```

运行结果：

```
[cTaskExecutor-1] com.xncoding.pos.mq.Receiver             : FANOUT_QUEUE_A "同学们集合啦！"
[cTaskExecutor-1] com.xncoding.pos.mq.Receiver             : DIRECT "定点消息"
[cTaskExecutor-1] com.xncoding.pos.mq.Receiver             : FANOUT_QUEUE_B "同学们集合啦！"
```

可以看到，广播消息被两个队列收到了，Direct消息只有一个收到。完美！

## GitHub源码

[springboot-rabbitmq](https://github.com/yidao620c/SpringBootBucket/tree/master/springboot-rabbitmq)

