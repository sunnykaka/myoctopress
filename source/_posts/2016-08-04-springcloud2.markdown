---
layout: post
title: "微服务框架Spring Cloud介绍 Part2: Spring Cloud与微服务"
date: 2016-08-03 22:09:25 +0800
comments: true
categories: [微服务, Spring Cloud]
description: "微服务框架Spring Cloud介绍 Part2: Spring Cloud与微服务"
---

之前介绍过[微服务的概念与Finagle框架](http://skaka.me/blog/2016/03/19/finagle1/), 这个系列介绍Spring Cloud.

Spring Cloud还是一个相对较新的框架, 今年(2016)才推出1.0的release版本. 虽然Spring Cloud时间最短, 但是相比我之前用过的Dubbo和Finagle, Spring Cloud提供的功能最齐全.

Spring Cloud完全依赖于Spring Boot, 我先简单介绍下Spring Boot.
[Spring Boot](http://projects.spring.io/spring-boot/)是Pivotal在Spring基础上推出的一个支持快速开发的框架. 如果是新项目, 建议基于Spring Boot而不是Spring.
以前使用Spring的项目, 需要自己指定一大堆项目依赖, 例如依赖Spring Core, Spring MVC, Mybatis等等, Spring Boot将这些依赖都模块化好了, 你不再需要自己手动去添加多个依赖项.
另外Spring Boot默认内嵌了一个Servlet容器, 你的页面可以直接通过main方法启动访问了, 不再需要部署到单独的应用服务器中, 这样应用的开发调试都会方便很多.
Spring Boot的这些特点使得它比较适合用来做微服务的基础框架, 但是要开发一个完整的微服务系统可不仅仅是从命令行启动一个web系统这么简单. Pivotal看到了这点, 推出了Spring Cloud.

[Spring Cloud](http://projects.spring.io/spring-cloud/)基于Spring Boot, 由众多的子项目组成. 例如[Spring Cloud Config](http://cloud.spring.io/spring-cloud-config)是一个中心化的配置管理服务,
用来解决微服务环境下配置文件分散管理的难题, [Spring Cloud Stream](http://cloud.spring.io/spring-cloud-stream)是一个消息中间件抽象层, 目前支持Redis, Rabbit MQ和Kafka,
[Spring Cloud Netflix](http://cloud.spring.io/spring-cloud-netflix)整合了[Netflix OSS](https://netflix.github.io/), 可以直接在项目中使用Netflix OSS.
目前Spring Cloud的子项目有接近20个, 如果要使用Spring Cloud, 务必先将子项目都了解一遍, 得知道哪些功能Spring Cloud已经提供了, 避免团队花费大量时间重复造轮子.

Spring Cloud是伴随着微服务的概念诞生的. 毫无疑问, 微服务真正落地是一项艰巨的任务. 不但是技术的变革, 也是开发方式的转变. 仅仅依靠Dubbo或Spring Cloud开发几个互相调用的服务不能算做是微服务.
一个合格的微服务系统必然包括从设计(从业务层面划分服务, 独立数据库), 到开发(选用合适的架构和工具, 解决CAP问题), 到测试(持续集成, 自动化测试), 到运维(容器化, 服务监控, 服务容错)的一系列解决方案.

我这个系列的博客就是介绍如何借助Spring Cloud和Netflix OSS, 来解决上面提到的问题.
之后的博客主要会涉及下面这些技术:  
**使用eureka和Netflix Ribbon进行服务注册和服务发现**  
**使用Spring Cloud Stream, zookeeper和kafka实现分布式事务**  
**使用hystrix实现服务隔离, 并且用hystrix dashboard和turbine监控hystrix服务**  
**使用Spring MVC和Swagger实现REST API**  
**使用Spring Cloud Config实现配置集中管理**  
**使用Spring Cloud Sleuth与Zipkin实现服务监控**  
...  

内容比较多, 我会分成多篇博客. 我不想泛泛地谈概念, 这样有点无趣, 对实际工作也起不到什么帮助.
我为演示这些技术的使用, 搭建了一个项目: mysteam.
我选择了一个简单的问题域, 电商系统里最基础的下单功能. 围绕下单功能, 系统拆分成了五个服务:  
**用户服务(user service)**  
**账户服务(account service)**  
**产品服务(product service)**  
**优惠券服务(coupon service)**  
**订单服务(order service)**  
下面是mysteam的架构示意图:
{% img /images/custom/20160804/mysteam_arch.png %}
我们的关注点主要在Backend Services和MQ, MySQL这一部分. 服务之间通过Rest API和事件进行通信. Rest API主要用来进行一些只读等不需要事务的操作,
涉及事务的操作一般使用事件来完成. 具体怎么做后面有专门的博客来介绍.

首先, 让我们来个Hello World, 先介绍如何将mysteam下载下来并启动.
一旦涉及微服务, 项目结构和环境都会比较复杂, 我已经尽量简化了, 请系好安全带: )

####1. 环境准备
**JDK 8+**  
**MySQL**  
**[kafka 0.8.22](https://www.apache.org/dyn/closer.cgi?path=/kafka/0.8.2.2/kafka_2.11-0.8.2.2.tgz)**  
**zookeeper** (可以下载, 也可以直接使用kafka自带的zookeeper)  
**Intellij IDEA或Eclipse** (这个项目结构比较复杂, IDE能起到很大帮助)  

<!-- more -->

####2. 从GitHub上下载项目
需要下载两个项目:  
**[mysteam](https://github.com/sunnykaka/mysteam)**  
**[mysteam-config-repo](https://github.com/sunnykaka/mysteam-config-repo)**  
mysteam是主项目, mysteam-config-repo是配置文件存放仓库, 后面讲Spring Cloud Config的时候会用到.

####3. 修改配置文件
######1. 修改配置文件读取路径
假设你的mysteam-config-repo项目存放路径是`D:/mysteam-config-repo`,
打开`$YOUR_PATH/mysteam/config/src/main/resources/application.yml`, 找到`uri: https://github.com/sunnykaka/mysteam-config-repo`这一行,
替换为`uri: file:///D:/mysteam-config-repo`.(如果你是linux系统, 并且mysteam-config-repo项目存放路径是`/home/my/mysteam-config-repo`,
则改为`uri: file:///home/my/mysteam-config-repo`).

######2. 修改kafka和zookeeper地址
打开`$YOUR_PATH/mysteam/config/src/main/resources/application.yml`, 将`brokers: 192.168.239.129:9092,192.168.239.129:9093,192.168.239.129:9094`修改成你的
kafka地址, 将`zkNodes: 192.168.239.129:2181`修改成你的zookeeper地址.
打开`$YOUR_PATH/mysteam-config-repo/application.yml`, 同样, 将`brokers: 192.168.239.129:9092,192.168.239.129:9093,192.168.239.129:9094`修改成你的
kafka地址, 将`zkNodes: 192.168.239.129:2181`修改成你的zookeeper地址.

######3. 修改MySQL数据库地址
进入`$YOUR_PATH/mysteam-config-repo`目录, 打开`account.yml`, `coupon.yml`, `order.yml`, `product.yml`, `user.yml`这几个文件,
找到`datasource`的配置, 将数据库的ip地址和端口, 以及用户名和密码修改成你的配置.

######4. 初始化数据库
数据库初始化文件是`$YOUR_PATH/mysteam/docs/init_database.sql`. 执行方法(假设你的mysteam目录是`D:/mysteam`, 数据库在本机3306, 用户名密码都是root):
```
cd D:/mysteam
mysql -uroot -proot < docs/init_database.sql
```
执行完成之后, 进入数据库应该可以看见5个数据库已经初始化好了.

####5. 启动Eureka服务, Config服务, 并运行测试.
主要介绍如何在IDE中启动服务.
因为Eureka和Config服务被其他服务使用, 所以要首先启动这两个服务. 其中Eureka服务要最先启动.
######1. 启动Eureka服务, 运行在1111端口
打开`$YOUR_PATH/mysteam/eureka/src/main/java/com/akkafun/eureka/EurekaApplication.java`, 直接运行main方法.
######2. 启动Config服务, 运行在8888端口.
打开`$YOUR_PATH/mysteam/config/src/main/java/com/akkafun/config/ConfigApplication.java`, 直接运行main方法.
######3. 运行EventBusTest测试.
打开`$YOUR_PATH/mysteam/user/core/src/test/java/com/akkafun/common/event/service/EventBusTest.java`, 运行junit测试.

这个测试的运行时间稍长, 在我机器上需要3分钟左右. 如果测试全部通过, 代表环境OK了.
如果运行报错, 则检查下前面的步骤看看问题出在哪儿. 特别关注下kafka和zookeeper的服务是不是启动了, 并且ip是否正确.

下一篇我会介绍mysteam的maven项目结构, 以及实现用户注册功能.
