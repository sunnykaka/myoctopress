---
layout: post
title: "微服务框架Spring Cloud介绍 Part6: 使用Spring Cloud Stream与Kafka"
date: 2016-09-13 20:03:22 +0800
comments: true
categories: [微服务, Spring Cloud]
description: "微服务框架Spring Cloud介绍 Part6: 使用Spring Cloud Stream与Kafka"
---

在微服务架构中, 系统之间主要通过两种方式交互: 同步的请求响应模式以及异步的消息模式. 如果微服务之间是通过异步的消息模式来进行数据交换, 一般会通过MQ进行.
 这篇文章中, 我会简单介绍Kafka和Spring Cloud Stream的基本概念, 并且通过代码演示如何使用Spring Cloud Stream与Kafka进行交互.

```
MQ是最重要的基础设施之一. mysteam中各个微服务的数据操作都会以事件的形式发送给MQ, 并由订阅了该事件的微服务读取事件进行消费. 这篇文章中, 我会简单介绍
Kafka和Spring Cloud Stream的基本概念, 并且通过代码和配置文件演示如何使用Spring Cloud Stream与Kafka进行交互.

在mysteam中, 微服务之间的交互方式主要有两种: 数据查询通过REST API进行, 数据更新会以事件的形式发送到MQ. 此外,


在mysteam项目中, MQ是最重要的基础设施之一. mysteam中各个微服务的数据操作都会以事件的形式发送给MQ, 并由订阅了该事件的微服务读取事件进行消费. 这篇文章中, 我会简单介绍
Kafka和Spring Cloud Stream的基本概念, 并且通过代码和配置文件演示如何使用Spring Cloud Stream与Kafka进行交互.
```

####1. Kafka简介
因为之后的代码和配置文件很多地方都需要涉及到Kafka中的概念, 所以我简单介绍下Kafka中的术语. 对Kafka已经比较熟悉的同学可以略过这一部分.
不同于传统的MQ, Kafka从设计之初就是一个分布式的消息系统. Kafka中有Producer和Consumer这两个角色, 顾名思义就是生产者和消费者. Producer负责将消息发送到对应的Topic下, 这里的Topic
和JMS中的Topic含义基本相同, 就是一个主题. Consumer负责从Topic中消费消息. 到这里看起来好像Kafka和传统的MQ编程模型是一样, 其实不然. 下面介绍Kafka特有的概念.  
**Broker**  
一个Broker相当于一个Kafka进程. 一般来说一个虚拟机上只会部署一个Kafka进程, 你可以理解为一个Broker就是一个Kafka服务器.  
**Partition**  
分区. 每个Topic会包含一个或多个分区. 分区的主要作用是用来做数据分片, 在你创建一个Topic的时候, 你可以为这个Topic指定分区的数量, 以及Topic中的消息按照什么字段来进行分区.
下面我举个例子来说明Partition, Broker以及Producer他们之间的关系. 假设我的Kafka集群有A,B,C这3个Broker, 我新建了一个名称为ORDER的Topic, 并且设定分区数量为3,
这3个分区可能会分别分布在A, B, C这三个Broker中. 我同时设置了ORDER这个Topic按照消息的id字段来分区, 这样当Producer准备发送一条消息到Topic的时候,
Kafka的客户端类库会根据我们设置的分区规则, 从消息中取出id字段然后取模分区数量来计算这条消息要发往哪个分区. 由此可见, 分区有点类似于关系数据库的水平拆分.
此外, 每个分区内是写入是完全按照顺序来的, 如果你想一个Topic内的消息完全有序, 则你只能有一个分区.  
**Consumer Group**  
消费组. 每个consumer属于一个特定的消费组, 在一个消费组内, 只能对Topic的消息消费一次. 还是举例说明, 假设c1, c2同属于一个消费组, 此时Topic内有一条消息m1. 假如c1已经读取了m1,
则c2就读取不到m1了. 但是多个消费组可以消费同一条消息. 比如我有另一个消费者c3属于另一个消费组, c3可以读取到m1. 简单来说, Topic内的消息, 对于不同消费组来说就像发布订阅的形式,
对于同一个消费组则是队列的形式.  
**Replication**  
分区的备份. 通过指定replication-factor, 分区可以分布到多台机器上. 例如replication-factor设置为3, 则分区会分布在A, B, C三台机器上. 其中假设A为leader, B, C为follower, 读写都会落到A上.  

####1. Spring Cloud Stream简介
Spring Cloud Stream(SCS)依赖于[Spring Integration](https://projects.spring.io/spring-integration/)这个库,
Spring Integration是对企业集成模式(Enterprise Integration Patterns)的实现与扩展(如果不知道什么是企业集成模式, 可以看看[这本书](https://book.douban.com/subject/1766652/)),
SCS直接使用了SI的组件来连接特定的MQ, 例如RabbitMQ, Kafka, 所以你在配置SCS的时候会发现很多SI的组件能够直接使用. SCS在SI的基础上, 对MQ的分布式特性做出了进一步的支持, 例如消费者分组和分区.
SCS应用结构如下如图.
{% img /images/custom/20161224/scs.png %}

Binder是SCS设计的一层MQ的抽象, 对不同的MQ有不同的实现. 例如连接Kafka的就叫Kafka Binder, 目前官方实现了的Binder有Kafka, Rabbit MQ和Redis.  
Input是消息的输入通道, 我们的应用从Input读取消息, Output相反. 关于Input和Output我们可以在之后的代码中看见实际例子.

####1. 在项目中使用Spring Cloud Stream


我们来看一个Spring Cloud Stream在代码中实际使用的例子:
```java
@EnableBinding(Processor.class)
@DependsOn("bindingService")
public class EventActivator {
    private static Logger logger = LoggerFactory.getLogger(EventActivator.class);

    //...

    @ServiceActivator(inputChannel = Processor.INPUT)
    public void receiveMessage(Object payload) {
        //TODO 验证receiveMessage是否是被spring顺序执行的
        byte[] bytes = (byte[]) payload;
        String message = new String(bytes, Charset.forName("UTF-8"));
        System.out.println(message);
    }
    //...
}
```
1.`Processor`定义了名称为INPUT和OUTPUT的两个MessageChannel, `@EnableBinding(Processor.class)`注解是告诉Spring启用Spring Cloud Stream
