---
title: RabbiMQ原理与SpringBoot使用
date: 2019-05-31 16:03:47
tags:
---
# RabbiMQ介绍

## 一、使用场景
RabbitMQ是一个消息中间件，所以最主要的作用就是：信息缓冲区，实现应用程序的异步和解耦。

RabbitMQ是实现AMQP（高级消息队列协议）的消息中间件的一种，最初起源于金融系统，用于在分布式系统中存储转发消息，在易用性、扩展性、高可用性等方面表现不俗。RabbitMQ主要是为了实现系统之间的双向解耦而实现的。当生产者大量产生数据时，消费者无法快速消费，那么需要一个中间层。保存这个数据。

AMQP，即Advanced Message Queuing Protocol，高级消息队列协议，是应用层协议的一个开放标准，为面向消息的中间件设计。消息中间件主要用于组件之间的解耦，消息的发送者无需知道消息使用者的存在，反之亦然。AMQP的主要特征是面向消息、队列、路由（包括点对点和发布/订阅）、可靠性、安全。详细概念可以参考官方指南 RabbitMQ

## 二、相关概念
通常我们谈到队列服务, 会有三个概念： 发消息者、队列、收消息者，RabbitMQ 在这个基本概念之上, 多做了一层抽象, 在发消息者和 队列之间, 加入了交换器 (Exchange). 这样发消息者和队列就没有直接联系, 转而变成发消息者把消息给交换器, 交换器根据调度策略再把消息再给队列。

那么，其中比较重要的概念有 4 个，分别为：虚拟主机，交换机，队列，和绑定。

- 虚拟主机v-host：一个虚拟主机持有一组交换机、队列和绑定。为什么需要多个虚拟主机呢？很简单，RabbitMQ当中，用户只能在虚拟主机的粒度进行权限控制。 因此，如果需要禁止A组访问B组的交换机/队列/绑定，必须为A和B分别创建一个虚拟主机。每一个RabbitMQ服务器都有一个默认的虚拟主机。
- 交换机：Exchange 用于转发消息，但是它不会做存储 ，如果没有 Queue bind 到 Exchange 的话，它会直接丢弃掉 Producer 发送过来的消息。这里有一个比较重要的概念：路由键 。消息到交换机的时候，交互机会转发到对应的队列中，那么究竟转发到哪个队列，就要根据该路由键。
- 绑定：也就是交换机需要和队列相绑定

### 交换机（Exchange）
交换机的功能主要是接收消息并且转发到绑定的队列，交换机不存储消息，在启用ack模式后，交换机找不到队列会返回错误。交换机有四种类型：Direct, topic, Headers and Fanout

- Direct：direct 类型的行为是"先匹配, 再投送". 即在绑定时设定一个 routing_key, 消息的routing_key 匹配时, 才会被交换器投送到绑定的队列中去.
- Topic：按规则转发消息（最灵活）
- Headers：设置header attribute参数类型的交换机
- Fanout：转发消息到所有绑定队列
### Direct Exchange
Direct Exchange是RabbitMQ默认的交换机模式，也是最简单的模式，根据key全文匹配去寻找队列。

![17987782-e1c8828c4d72ceef](17987782-e1c8828c4d72ceef.png)

第一个 X - Q1 就有一个 binding key，名字为 orange； X - Q2 就有 2 个 binding key，名字为 black 和 green。当消息中的 路由键 和 这个 binding key 对应上的时候，那么就知道了该消息去到哪一个队列中。

Ps：为什么 X 到 Q2 要有 black，green，2个 binding key呢，一个不就行了吗？ - 这个主要是因为可能又有 Q3，而Q3只接受 black 的信息，而Q2不仅接受black 的信息，还接受 green 的信息。

### Topic Exchange
根据通配符转发消息到队列，在这种交换机下，队列和交换机的绑定会定义一种路由模式，那么，通配符就要在这种路由模式和路由键之间匹配后交换机才能转发消息。

- *（星号）可以替代一个单词。
- ＃（hash）可以替换零个或多个单词。

![17987782-c8db36821b6fcd30](17987782-c8db36821b6fcd30.png)

### Headers Exchange
headers 也是根据规则匹配, 相较于 direct 和 topic 固定地使用 routing_key , headers 则是一个自定义匹配规则的类型.
在队列与交换器绑定时, 会设定一组键值对规则, 消息中也包括一组键值对( headers 属性), 当这些键值对有一对, 或全部匹配时, 消息被投送到对应队列.

### Fanout Exchange
消息广播的模式，也就是我们的发布订阅模式。Fanout Exchange 消息广播的模式，不管路由键或者是路由模式，会把消息发给绑定给它的全部队列，如果配置了routing_key会被忽略。

### 消息确认
消息消费者如何通知 Rabbit 消息消费成功？

消息通过 ACK 确认是否被正确接收，每个 Message 都要被确认（acknowledged），可以手动去 ACK 或自动 ACK 自动确认会在消息发送给消费者后立即确认，但存在丢失消息的可能，如果消费端消费逻辑抛出异常，也就是消费端没有处理成功这条消息，那么就相当于丢失了消息 如果消息已经被处理，但后续代码抛出异常，使用 Spring 进行管理的话消费端业务逻辑会进行回滚，这也同样造成了实际意义的消息丢失 如果手动确认则当消费者调用 ack、nack、reject 几种方法进行确认，手动确认可以在业务失败后进行一些操作，如果消息未被 ACK 则会发送到下一个消费者 如果某个服务忘记 ACK 了，则 RabbitMQ 不会再发送数据给它，因为 RabbitMQ 认为该服务的处理能力有限 ACK 机制还可以起到限流作用，比如在接收到某条消息时休眠几秒钟 消息确认模式有：

- AcknowledgeMode.NONE：自动确认
- AcknowledgeMode.AUTO：根据情况确认
- AcknowledgeMode.MANUAL：手动确认

## SpringBoot集成RabbitMQ
配置pom，主要添加spring-boot-starter-amqp支持，springboot基于2.1.4版本

``` java
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-amqp</artifactId>
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
<dependency>
    <groupId>org.projectlombok</groupId>
    <artifactId>lombok</artifactId>
    <optional>true</optional>
</dependency>
```

+ 配置springboot的yaml文件

``` java
server:
  servlet:
    context-path: /rabbitmq
  port: 9004
spring:
  application:
    name: rabbitmq
  rabbitmq:
    host: localhost
    virtual-host: /crawl
    username: xxxx
    password: xxx
    port: 5672
    # 消息失败返回，比如路由不到队列时触发回调
    publisher-returns: true
    # 消息正确发送确认
    publisher-confirms: true
    template:
      retry:
        enabled: true
        initial-interval: 2s
    listener:
      simple:
        # 手动ACK 不开启自动ACK模式,目的是防止报错后未正确处理消息丢失 默认 为 none
        acknowledge-mode: manual
```

另外我们还要配置ACK确认回调的配置，通过实现RabbitTemplate.ConfirmCallback接口，消息发送到Broker后触发回调，也就是只能确认是否正确到达Exchange中。

``` java
import lombok.extern.slf4j.Slf4j;
import org.springframework.amqp.rabbit.connection.CorrelationData;
import org.springframework.amqp.rabbit.core.RabbitTemplate;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

import javax.annotation.PostConstruct;

@Component
@Slf4j
public class RabbitTemplateConfirmCallback implements RabbitTemplate.ConfirmCallback {
    @Autowired
    private RabbitTemplate rabbitTemplate;

    @PostConstruct
    public void init() {
        //指定 ConfirmCallback
        rabbitTemplate.setConfirmCallback(this);
    }

    @Override
    public void confirm(CorrelationData correlationData, boolean ack, String cause) {
        log.info("消息唯一标识：{},确认结果：{},失败原因：{}", correlationData, ack, cause);
    }
}
```


消息失败返回，比如路由步到队列就会触发，如果西区奥西发送到交换器成功，但是没有匹配的队列就会触发回调

``` java
import lombok.extern.slf4j.Slf4j;
import org.springframework.amqp.core.Message;
import org.springframework.amqp.rabbit.core.RabbitTemplate;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

import javax.annotation.PostConstruct;

@Component
@Slf4j
public class RabbitTemplateReturnCallback implements RabbitTemplate.ReturnCallback {
    @Autowired
    private RabbitTemplate rabbitTemplate;

    @PostConstruct
    public void init() {
        //指定 ReturnCallback
        rabbitTemplate.setReturnCallback(this);
        rabbitTemplate.setMandatory(true);
    }

    @Override
    public void returnedMessage(Message message, int replyCode, String replyText, String exchange, String routingKey) {
        log.info("消息主体 message : " + message);
        log.info("消息主体 message : " + replyCode);
        log.info("描述：" + replyText);
        log.info("消息使用的交换器 exchange : " + exchange);
        log.info("消息使用的路由键 routing : " + routingKey);
    }
}

```

### 一、简单的开始-简单队列
如下图：

“P”是我们的生产者，“C”是我们的消费者。中间的框是一个队列 - RabbitMQ代表消费者保留的消息缓冲区。

![17987782-422f423ccc1137f6](17987782-422f423ccc1137f6.png)

新增SimpleConfig，创建我们要投放的队列：代码如下:

``` java
 /**
  * 队列直接投放
  */
 @Configuration
 public class SimpleConfig {
     @Bean
     public Queue simpleQueue() {
         return new Queue("simple");
     }
 }
```

再分别创建消息发送者与消息接收者：

- 消息发送者

``` java
import lombok.extern.slf4j.Slf4j;
import org.springframework.amqp.rabbit.connection.CorrelationData;
import org.springframework.amqp.rabbit.core.RabbitTemplate;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;
import zero.springboot.study.rabbitmq.model.User;

import java.util.UUID;

@Component
@Slf4j
public class HelloSender {
    @Autowired
    private RabbitTemplate rabbitTemplate;

    public void send() {
        User user = new User();
        user.setName("青");
        user.setPass("111111");
        //发送消息到hello队列
        log.info("发送消息：{}", user);
        rabbitTemplate.convertAndSend("hello", user, new CorrelationData(UUID.randomUUID().toString()));

        String msg = "hello qing";
        log.info("发送消息：{}", msg);
        rabbitTemplate.convertAndSend("simple", msg);
    }
}
```

- 消息接收者

``` java
import com.rabbitmq.client.Channel;
import lombok.extern.slf4j.Slf4j;
import org.springframework.amqp.rabbit.annotation.RabbitHandler;
import org.springframework.amqp.rabbit.annotation.RabbitListener;
import org.springframework.amqp.support.AmqpHeaders;
import org.springframework.messaging.handler.annotation.Header;
import org.springframework.stereotype.Component;
import zero.springboot.study.rabbitmq.model.User;

import java.io.IOException;

@Component
@Slf4j
@RabbitListener(queues = "simple")
public class HelloReceiver {

    @RabbitHandler
    public void processUser(User user, Channel channel, @Header(AmqpHeaders.DELIVERY_TAG) long tag) {
        log.info("收到消息：{}", user);
        // 手动ACK
        try {
//            //消息确认，代表消费者确认收到当前消息，语义上表示消费者成功处理了当前消息。
            channel.basicAck(tag, false);
//             代表消费者拒绝一条或者多条消息，第二个参数表示一次是否拒绝多条消息，第三个参数表示是否把当前消息重新入队
//        channel.basicNack(deliveryTag, false, false);

            // 代表消费者拒绝当前消息，第二个参数表示是否把当前消息重新入队
//        channel.basicReject(deliveryTag,false);
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    @RabbitHandler
    public void processString(String message, Channel channel, @Header(AmqpHeaders.DELIVERY_TAG) long tag) {
        log.info("收到消息：{}", message);
        // 手动ACK
        try {
//            //消息确认，代表消费者确认收到当前消息，语义上表示消费者成功处理了当前消息。
            channel.basicAck(tag, false);
//             代表消费者拒绝一条或者多条消息，第二个参数表示一次是否拒绝多条消息，第三个参数表示是否把当前消息重新入队
//        channel.basicNack(deliveryTag, false, false);

            // 代表消费者拒绝当前消息，第二个参数表示是否把当前消息重新入队
//        channel.basicReject(deliveryTag,false);
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

这样就实现了简单的消息发送到指定队列的模式。我们写一个测试类

### 二、Direct Exchange模式
主要配置我们的Direct Exchange交换机，并且创建队列通过routing key 绑定到交换机上

``` java
import org.springframework.amqp.core.Binding;
import org.springframework.amqp.core.BindingBuilder;
import org.springframework.amqp.core.DirectExchange;
import org.springframework.amqp.core.Queue;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class DirectConfig {

    //队列名字
    public static final String QUEUE_NAME = "direct_name";

    //交换机名称
    public static final String EXCHANGE = "zero-exchange";

    //路由键名称
    public static final String ROUTING_KEY = "routingKey";

    @Bean
    public Queue blueQueue() {
        return new Queue(QUEUE_NAME, true);
    }

    @Bean
    public DirectExchange defaultExchange() {
        return new DirectExchange(EXCHANGE);
    }

    @Bean
    public Binding bindingBlue() {
        return BindingBuilder.bind(blueQueue()).to(defaultExchange()).with(ROUTING_KEY);
    }

}
```
接下来我们创建生产者与消费者

- 生产者:

``` java
import lombok.extern.slf4j.Slf4j;
import org.springframework.amqp.rabbit.connection.CorrelationData;
import org.springframework.amqp.rabbit.core.RabbitTemplate;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;
import zero.springboot.study.rabbitmq.config.DirectConfig;
import zero.springboot.study.rabbitmq.model.User;

import java.util.UUID;

@Component
@Slf4j
public class DirectSender {
    @Autowired
    private RabbitTemplate rabbitTemplate;

    public void send() {
        User user = new User();
        user.setName("青");
        user.setPass("111111");
        //发送消息到hello队列
        log.info("DirectReceiver发送消息：{}", user);
        rabbitTemplate.convertAndSend(DirectConfig.EXCHANGE, DirectConfig.ROUTING_KEY, user, new CorrelationData(UUID.randomUUID().toString()));

        String msg = "hello qing";
        log.info("DirectReceiver发送消息：{}", msg);
        rabbitTemplate.convertAndSend(DirectConfig.EXCHANGE, DirectConfig.ROUTING_KEY, msg);
    }
}

```

- 消费者:

``` java

@Component
@Slf4j
@RabbitListener(queues = "direct_name")
public class DirectReceiver {

    @RabbitHandler
    public void processUser(User user, Channel channel, @Header(AmqpHeaders.DELIVERY_TAG) long tag) {
        log.info("DirectReceiver收到消息：{}", user);
        // 手动ACK
        try {
//            //消息确认，代表消费者确认收到当前消息，语义上表示消费者成功处理了当前消息。
            channel.basicAck(tag, false);
//             代表消费者拒绝一条或者多条消息，第二个参数表示一次是否拒绝多条消息，第三个参数表示是否把当前消息重新入队
//        channel.basicNack(deliveryTag, false, false);

            // 代表消费者拒绝当前消息，第二个参数表示是否把当前消息重新入队
//        channel.basicReject(deliveryTag,false);
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    @RabbitHandler
    public void processString(String message, Channel channel, @Header(AmqpHeaders.DELIVERY_TAG) long tag) {
        log.info("收到消息：{}", message);
        // 手动ACK
        try {
//            //消息确认，代表消费者确认收到当前消息，语义上表示消费者成功处理了当前消息。
            channel.basicAck(tag, false);
//             代表消费者拒绝一条或者多条消息，第二个参数表示一次是否拒绝多条消息，第三个参数表示是否把当前消息重新入队
//        channel.basicNack(deliveryTag, false, false);

            // 代表消费者拒绝当前消息，第二个参数表示是否把当前消息重新入队
//        channel.basicReject(deliveryTag,false);
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}

```

### 三、Topic Exchange模式
创建队列以及交换机。并通过路由匹配规则将队列与交换机绑定上

``` java
import org.springframework.amqp.core.Binding;
import org.springframework.amqp.core.BindingBuilder;
import org.springframework.amqp.core.Queue;
import org.springframework.amqp.core.TopicExchange;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;


@Configuration
public class TopicRabbitConfig {
    final static String message = "topic.message";
    final static String messages = "topic.messages";

    @Bean
    public Queue queueMessage() {
        return new Queue(TopicRabbitConfig.message);
    }

    @Bean
    public Queue queueMessages() {
        return new Queue(TopicRabbitConfig.messages);
    }

    @Bean
    TopicExchange exchange() {
        return new TopicExchange("topicExchange");
    }

    @Bean
    Binding bindingExchangeMessage(@Qualifier("queueMessage") Queue queueMessage, TopicExchange exchange) {
        return BindingBuilder.bind(queueMessage).to(exchange).with("topic.message");
    }

    @Bean
    Binding bindingExchangeMessages(@Qualifier("queueMessages") Queue queueMessages, TopicExchange exchange) {
        return BindingBuilder.bind(queueMessages).to(exchange).with("topic.#");
    }

}
```

- 创建生产者

``` java

import lombok.extern.slf4j.Slf4j;
import org.springframework.amqp.rabbit.core.RabbitTemplate;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;


@Component
@Slf4j
public class TopicSender {
    @Autowired
    private RabbitTemplate rabbitTemplate;

    /**
     * 匹配topic.message，两个队列都会收到
     */
    public void send1() {
        String context = "hi, i am message 1";
        log.info("主题发送 : {}" , context);
        rabbitTemplate.convertAndSend("topicExchange", "topic.message", context);
    }

    /**
     * 匹配topic.messages
     */
    public void send2() {
        String context = "hi, i am messages 2";
        log.info("主题发送 : {}" , context);
        rabbitTemplate.convertAndSend("topicExchange", "topic.messages", context);
    }
}

```

- 创建消费者，这里我们分别创建两个队列的消费者

``` java 
@Component
@RabbitListener(queues = "topic.message")
@Slf4j
public class TopicReceiver {

    @RabbitHandler
    public void process(String message, Channel channel, @Header(AmqpHeaders.DELIVERY_TAG) long tag) {
        log.info("topic.message Receiver1  {}: ", message);
        // 手动ACK
        try {
//            //消息确认，代表消费者确认收到当前消息，语义上表示消费者成功处理了当前消息。
            channel.basicAck(tag, false);
//             代表消费者拒绝一条或者多条消息，第二个参数表示一次是否拒绝多条消息，第三个参数表示是否把当前消息重新入队
//        channel.basicNack(deliveryTag, false, false);

            // 代表消费者拒绝当前消息，第二个参数表示是否把当前消息重新入队
//        channel.basicReject(deliveryTag,false);
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

}
```

第二个消费者

``` java
@Component
@RabbitListener(queues = "topic.messages")
@Slf4j
public class TopicReceiver2 {

    @RabbitHandler
    public void process(String message, Channel channel, @Header(AmqpHeaders.DELIVERY_TAG) long tag) {
        log.info("topic.messages Receiver2  : {}", message);
        // 手动ACK
        try {
//            //消息确认，代表消费者确认收到当前消息，语义上表示消费者成功处理了当前消息。
            channel.basicAck(tag, false);
//             代表消费者拒绝一条或者多条消息，第二个参数表示一次是否拒绝多条消息，第三个参数表示是否把当前消息重新入队
//        channel.basicNack(deliveryTag, false, false);

            // 代表消费者拒绝当前消息，第二个参数表示是否把当前消息重新入队
//        channel.basicReject(deliveryTag,false);
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

}
```

### 四、Fanout 模式
也就是发布、订阅。所有绑定在交换机上的队列都会收到消息，发送端指定的routing key的任何字符都会被忽略

配置交换机与队列


``` java
@Configuration
public class FanoutRabbitConfig {
    @Bean
    public Queue AMessage() {
        return new Queue("fanout.A");
    }

    @Bean
    public Queue BMessage() {
        return new Queue("fanout.B");
    }

    @Bean
    public Queue CMessage() {
        return new Queue("fanout.C");
    }

    @Bean
    FanoutExchange fanoutExchange() {
        return new FanoutExchange("fanoutExchange");
    }
    @Bean
    Binding bindingExchangeA(Queue AMessage, FanoutExchange fanoutExchange) {
        return BindingBuilder.bind(AMessage).to(fanoutExchange);
    }

    @Bean
    Binding bindingExchangeB(Queue BMessage, FanoutExchange fanoutExchange) {
        return BindingBuilder.bind(BMessage).to(fanoutExchange);
    }

    @Bean
    Binding bindingExchangeC(Queue CMessage, FanoutExchange fanoutExchange) {
        return BindingBuilder.bind(CMessage).to(fanoutExchange);
    }
}
```

- 创建发送者

``` java
@Component
@Slf4j
public class FanoutSender {
    @Autowired
    private RabbitTemplate rabbitTemplate;

    public void send() {
        String context = "hi, fanout msg ";
        rabbitTemplate.convertAndSend("fanoutExchange", null, context);
    }

}
```

- 创建A、B、C队列消费者

``` java
@Component
@RabbitListener(queues = "fanout.A")
@Slf4j
public class FanoutReceiverA {

    @RabbitHandler
    public void process(String message, Channel channel, @Header(AmqpHeaders.DELIVERY_TAG) long tag) {
        log.info("fanout Receiver A  : {}" , message);
        // 手动ACK
        try {
//            //消息确认，代表消费者确认收到当前消息，语义上表示消费者成功处理了当前消息。
            channel.basicAck(tag, false);
//             代表消费者拒绝一条或者多条消息，第二个参数表示一次是否拒绝多条消息，第三个参数表示是否把当前消息重新入队
//        channel.basicNack(deliveryTag, false, false);

            // 代表消费者拒绝当前消息，第二个参数表示是否把当前消息重新入队
//        channel.basicReject(deliveryTag,false);
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

}
```

剩下的B、C就不重复贴了。

#### 单元测试:

``` java

import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.junit4.SpringRunner;
import zero.springboot.study.rabbitmq.direct.DirectSender;
import zero.springboot.study.rabbitmq.fanout.FanoutSender;
import zero.springboot.study.rabbitmq.simple.HelloSender;
import zero.springboot.study.rabbitmq.topic.TopicSender;

@RunWith(SpringRunner.class)
@SpringBootTest(classes = RabbitmqApplication.class)
public class RabbitmqApplicationTests {

    @Autowired
    private DirectSender directSender;

    @Autowired
    private TopicSender topicSender;

    @Autowired
    private FanoutSender fanoutSender;

    @Autowired
    private HelloSender helloSender;

    @Test
    public void testDirect() {
        directSender.send();
    }

    @Test
    public void topic1() {
        topicSender.send1();
    }

    @Test
    public void topic2() {
        topicSender.send2();
    }

    @Test
    public void testFanout() {
        fanoutSender.send();
    }

    @Test
    public void testSimple() {
        helloSender.send();
    }

}
```