---
layout: post
title: "微服务框架Spring Cloud介绍 Part5: 在微服务系统中使用Hystrix, Hystrix Dashboard与Turbine"
date: 2016-09-04 21:03:22 +0800
comments: true
categories: [微服务, Spring Cloud]
description: "微服务框架Spring Cloud介绍 Part5: 在微服务系统中使用Hystrix, Hystrix Dashboard与Turbine"
---

通过前面几篇文章的介绍, 我们已经能够使用Spring Cloud开发出一个简单的系统了. 这篇文章里, 我们将关注点转移到如何提高微服务系统的容错性(Fault Tolerance),
并了解如何借助Hystrix开发健壮的微服务.  

以往我们在开发单体应用, 或者调用RPC服务的时候, 可能没有考虑太多目标服务调用失败的情况, 经常一个Try/Catch加上打印日志就解决了.
但是在微服务系统中, 这种处理方法会给系统的稳定性带来很大隐患.举个例子, 假设我们系统中下单的功能要依赖50个服务, 每个服务正常响应的概率为99.99%,
如果我们不做容错处理, 只要任意一个服务没有响应下单就失败的话, 我们下单成功的概率为

>99.99<sup>50</sup>  =  99.5%

日订单量为1W的话, 50个会出现下单失败, 这还是建立在依赖服务稳定性很高的情况下(4个9). 但是服务调用失败引起的问题不仅仅是这么简单, 在分布式环境下,
一个服务的调用失败可能会使其他被依赖服务发生延迟和超时, 而且这个影响会很快扩散到其他服务, 从而引发整个系统的雪崩(Avalanche).

####1. hystrix介绍
这篇文章要介绍的Hystrix是一个Java类库, 它提供下面这些功能来帮助我们构建健壮的微服务系统:(对Hystrix已经比较熟悉的同学可以直接跳过这段到下面的Hystrix javanica介绍)  
**1.断路器机制**  
断路器很好理解, 当Hystrix Command请求后端服务失败数量超过一定比例(默认50%), 断路器会切换到开路状态(Open). 这时所有请求会直接失败而不会发送到后端服务.
断路器保持在开路状态一段时间后(默认5秒), 自动切换到半开路状态(HALF-OPEN). 这时会判断下一次请求的返回情况, 如果请求成功, 断路器切回闭路状态(CLOSED), 否则重新切换到开路状态(OPEN).
Hystrix的断路器就像我们家庭电路中的保险丝, 一旦后端服务不可用, 断路器会直接切断请求链, 避免发送大量无效请求影响系统吞吐量, 并且断路器有自我检测并恢复的能力.
**2.Fallback**  
Fallback相当于是降级操作. 对于查询操作, 我们可以实现一个fallback方法, 当请求后端服务出现异常的时候, 可以使用fallback方法返回的值. fallback方法的返回值一般是设置的默认值或者来自缓存.  
**3.资源隔离**  
在Hystrix中, 主要通过线程池来实现资源隔离. 通常在使用的时候我们会根据调用的远程服务划分出多个线程池. 例如调用产品服务的Command放入A线程池, 调用账户服务的Command放入B线程池.
这样做的主要优点是运行环境被隔离开了. 这样就算调用服务的代码存在bug或者由于其他原因导致自己所在线程池被耗尽时, 不会对系统的其他服务造成影响.
但是带来的代价就是维护多个线程池会对系统带来额外的性能开销. 如果是对性能有严格要求而且确信自己调用服务的客户端代码不会出问题的话, 可以使用Hystrix的信号模式(Semaphores)来隔离资源.  

以上是对Hystrix的简单介绍, 如果想进一步了解Hystrix可以访问[GitHub](https://github.com/Netflix/Hystrix/wiki). 现在我们来看如何编写一个Hystrix Command, 代码来自Hystrix的Github:
```java
public class CommandHelloFailure extends HystrixCommand<String> {

    private final String name;

    public CommandHelloFailure(String name) {
        super(Setter.withGroupKey(HystrixCommandGroupKey.Factory.asKey("ExampleGroup"))
                      .andCommandKey(HystrixCommandKey.Factory.asKey("HelloWorld"))
                      .andThreadPoolKey(HystrixThreadPoolKey.Factory.asKey("HelloWorldPool")));
        this.name = name;
    }

    @Override
    protected String run() {
        return "hello world";
    }

    @Override
    protected String getFallback() {
        return "Hello Failure " + name + "!";
    }
}
```
编写一个Hystrix Command, 需要继承`HystrixCommand`类, 在`run`方法中完成你的业务逻辑. 你可以重写`getFallback`方法来在run方法抛出异常的时候返回备用的结果.

####2. Hystrix javanica介绍

上面的代码写起来有点繁琐, 好在有[javanica](https://github.com/Netflix/Hystrix/tree/master/hystrix-contrib/hystrix-javanica). 这是Hystrix开源社区贡献的一个类库.
使用javanica, 你不需要继承`HystrixCommand`, 只需要上加上`@HystrixCommand`注解, 你的方法就能够通过Hystrix运行. 例如下面的代码:
```java
@HystrixCommand(groupKey="ExampleGroup", commandKey = "HelloWorld",
         threadPoolKey="HelloWorldPool", fallbackMethod = "defaultHello")
public String getUserById(String name) {
    return "hello world";
}

private String defaultHello(String name) {
    return "Hello Failure " + name + "!";
}
```
这段代码与之前的继承`HystrixCommand`代码所完成的工作是一样的, 如果习惯了使用Java注解和Spring, 一般会更习惯javanica的开发方式.

####3. 在Spring Cloud中使用Hystrix
了解了Hystrix和javanica, 现在我们来看看如何在基于Spring Cloud的项目中使用Hystrix.
#####1. 添加maven依赖
在pom.xml文件中加入下面的依赖
```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-hystrix</artifactId>
</dependency>
```
mysteam项目里, 依赖被加到了`$YOUR_PATH/mysteam/common/pom.xml`文件中.

#####2. Spring配置加上@EnableHystrix注解
`BaseConfiguration`类中加了这个注解, 文件位置在`$YOUR_PATH/mysteam/common/src/main/java/com/akkafun/common/spring/BaseConfiguration.java`.

#####3. 方法上添加@HystrixCommand注解
上篇文章中我们开发了下单的功能. 为了查询账户余额是否足够, 我们调用了`AccountGateway`的`isBalanceEnough`方法:
```java
@HystrixCommand(ignoreExceptions = RemoteCallException.class)
public boolean isBalanceEnough(Long userId, Long amount) {
    return accountClient.checkEnoughBalance(userId, amount).isSuccess();
}
```
这里我们没有指定`commandKey`和`groupKey`参数, `commandKey`的默认值是方法名称`isBalanceEnough`, `groupKey`的默认值是类名`AccountGateway`.
javanica使用AOP来完成普通方法到Hystrix Command的转换, 所以我们只需要在方法上加上`@HystrixCommand`注解, 就能让这个方法成为一个Hystrix Command了, 相当便捷.
这里说明一下`ignoreExceptions = RemoteCallException.class`这个配置的含义. 在Hystrix Command的`run`方法执行的时候, 如果抛出了`HystrixBadRequestException`异常,
是不会触发Fallback逻辑而是直接失败, 这个异常一般被用来提示客户端参数请求错误或者其他需要直接失败的错误.
```java
@HystrixCommand(ignoreExceptions = RemoteCallException.class)
```
这个配置的意思是, 如果方法中抛出了`RemoteCallException`(这个异常是我们自己定义的), javanica会负责将异常包装为`HystrixBadRequestException`并抛出.
这里指定`RemoteCallException`是因为在mysteam中, 远程服务返回的正常错误信息会被包装为`RemoteCallException`抛出, 因为是正常的返回, 我们不想让Hystrix执行Fallback逻辑,
所以在这里配置了ignoreExceptions参数(如果想了解mysteam的http请求和异常处理逻辑, 可以查看`CustomRestTemplate`类和`RestTemplateErrorHandler`类的代码).

####4. 在Spring Cloud中使用Hystrix Dashboard和Turbine
在生产环境中一般你需要对Hystrix Command的groupKey和线程池大小等参数进行调整, 而且针对生产环境, Netflix还给我们准备了一个非常好用的运维工具,
那就是[Hystrix Dashboard](https://github.com/Netflix/Hystrix/tree/master/hystrix-dashboard)和[Turbine](https://github.com/Netflix/Turbine).  
先上一张图有个直观感受, 来自Turbine的GitHub
{% img https://github.com/Netflix/Turbine/wiki/images/NetflixDash.jpg %}
通过Hystrix Dashboard我们可以在直观地看到各Hystrix Command的请求响应时间, 以及请求成功率等数据. 但是只使用Hystrix Dashboard的话, 你只能看到单个应用内的服务信息, 这明显不够.
我们需要一个工具能让我们汇总系统内多个服务的数据并显示到Hystrix Dashboard上, 这个工具就是Turbine.  

我在mysteam中已经加了一个turbine模块, 模块名称是`turbine`, 接下来, 让我们启动turbine服务.
#####1. 首先启动EurekaApplication和ConfigApplication
关于为什么要先启动这两个服务, 可以参看我[之前的文章](http://skaka.me/blog/2016/08/03/springcloud2/).
#####2. 启动turbine模块下的TurbineApplication
启动完成之后, 打开链接[http://localhost:7777/hystrix](http://localhost:7777/hystrix), 你应该能看见可爱的豪猪logo. 在页面中部的输入框输入你要监控的turbine流地址, 例如输入
`http://localhost:7777/turbine.stream?cluster=ORDER`, 代表我们要监控ORDER服务下所有实例的Hystrix. 输入完成点击`Monitor Stream`按钮, 你就能进入监控UI了. 但是这时应该还没有显示图表,
因为ORDER服务还没有启动. 到此为止, 如果你只是想启动Turbine服务, 那么你已经完成了. 但是如果你想看到实际的监控信息, 还需要启动其他服务并构造数据. 接下来让我们启动其他的5个服务.
#####3. 启动AccountApplication, CouponApplication, OrderApplication, ProductApplication, UserApplication
#####4. 运行integration-test模块下的OrderIntegrationTest.testCreateOrderSuccess方法生成测试数据
`OrderIntegrationTest`类的位置在`$YOUR_PATH/mysteam/integration-test/core/src/test/java/com/akkafun/integrationtest/order/OrderIntegrationTest.java`, 这是一个下单功能的集成测试类.
这个类会模拟真实用户下单, 所以运行这个测试类之前需要把依赖的服务全部启动好. `testCreateOrderSuccess`测试方法运行完成之后, 在浏览器里你应该能看见如下的界面
{% img /images/custom/20160904/turbine.png %}
这张图里, `Circuit`标签下显示的是断路器信息, 其中显示了ORDER服务的两个断路器, 分别是`findProducts`和`isBalanceEnough`, 并且状态都是`Closed`. `Thread Pools`标签下显示的是线程池的信息.
这张图里各参数的详细含义, 大家可以参考[Dashboard的Wiki](https://github.com/Netflix/Hystrix/wiki/Dashboard). 通过Hystrix Dashboard和Turbine, 我们能够很方便地监控每个Hystrix Command的运行情况,
在出现问题的时候能够及时定位到问题所在的服务. Tubine本质是一个数据聚合服务, 我们可以使用Turbine的数据开发一些定制的功能. 比如我之前开发的预警系统, 会实时对Turbine的流式数据进行消费,
在发现Hystrix Command调用失败次数达到一定阀值的时候, 会根据调用链定位到疑似的问题服务并发出告警. 你也可以很容易的将Hystrix Dashbord和Turbine整合到你自己的监控系统里.  

这一篇文章介绍的Hystrix和上一篇文章介绍的Eureka, Ribbon都属于Spring Cloud Netflix这个子项目的内容. 下一篇里我们将把目光移到分布式系统最重要的中间件: MQ,
了解如何使用Spring Cloud Stream与Kafka交互.
