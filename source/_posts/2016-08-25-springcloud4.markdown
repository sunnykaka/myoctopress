---
layout: post
title: "微服务框架Spring Cloud介绍 Part4: 使用Eureka, Ribbon, Feign实现REST服务客户端"
date: 2016-08-25 19:52:31 +0800
comments: true
categories: [微服务, Spring Cloud]
description: "微服务框架Spring Cloud介绍 Part4: 使用Eureka, Ribbon, Feign实现REST服务客户端"
---

在[上一篇文章](http://skaka.me/blog/2016/08/10/springcloud3/)中我们开发了一个用户注册服务. 这篇文章我将介绍如何开发mysteam订单服务中的下单功能,
下单功能会涉及服务之间的交互与事件的处理, 并且我会对开发过程中用到的框架和类库进行简单地讲解. 开始写代码之前, 我们先来看看下单的处理流程:
{% img /images/custom/20160825/place_order.png %}

其中1,2,3,4,11步的黑色箭头代表是同步操作, 5,6,7,8,9,10步是异步操作.
下单接口接收要订购的产品ID, 数量和要使用的优惠券ID, 然后调用产品服务的接口查询产品信息, 调用优惠券接口校验优惠券是否有效,
以及调用账户接口判断账户金额是否足够(mysteam是一个虚拟物品商城, 采用先充值后购买的形式). 如果这些校验都成功, 订单服务会发送账户扣款事件和优惠券使用事件到MQ,
账户服务和优惠券服务会从MQ读取事件进行处理, 如果处理成功, 订单服务将能接收到结果, 并且将订单状态置为`下单成功`, 如果处理失败或超时, 订单状态会被置为`下单失败`.  

####1. 实现Model
流程清楚了, 现在我们来看代码. 订单类是`$YOUR_PATH/mysteam/order/core/src/main/java/com/akkafun/order/domain/Order.java`, 订单类和前面的用户类类似, 其中有两个字段需要注意一下:
```java
@Column
@Enumerated(value = EnumType.STRING)
private OrderStatus status;

@OneToMany(fetch = FetchType.LAZY, mappedBy = "order")
private List<OrderItem> orderItemList = new ArrayList<>(0);
```
OrderStatus是一个枚举, 表示订单状态. OrderItem是订单项.  

<!-- more -->

####2. 实现DAO
DAO层基本没有实际的代码, 就不贴了.  

####3. 实现Service
下单的业务逻辑都在service内, 打开`$YOUR_PATH/mysteam/order/core/src/main/java/com/akkafun/order/service/OrderService.java`, 找到`placeOrder`方法:
```java
/**
 * 下订单
 *
 * @param placeOrderDto
 * @return
 */
@Transactional
public Order placeOrder(PlaceOrderDto placeOrderDto) {

    //...

    //#1
    //查询产品信息
    List<Long> productIds = placeOrderDto.getPlaceOrderItemList().stream()
            .map(PlaceOrderItemDto::getProductId)
            .collect(Collectors.toList());

    List<ProductDto> productDtoList = productGateway.findProducts(productIds);

    //...

    //#2
    //查询优惠券信息
    List<OrderCoupon> orderCouponList = new ArrayList<>();
    Set<Long> couponIdSet = new HashSet<>(placeOrderDto.getCouponIdList());
    if (!couponIdSet.isEmpty()) {

        //#2
        List<CouponDto> couponDtoList = couponGateway.findCoupons(new ArrayList<>(couponIdSet));

        orderCouponList = couponDtoList.stream().map(couponDto -> {
            OrderCoupon orderCoupon = new OrderCoupon();
            orderCoupon.setCouponAmount(couponDto.getAmount());
            orderCoupon.setCouponCode(couponDto.getCode());
            orderCoupon.setCouponId(couponDto.getId());
            return orderCoupon;
        }).collect(Collectors.toList());

    }

    //#3
    //计算订单金额
    long couponAmount = orderCouponList.stream().mapToLong(OrderCoupon::getCouponAmount).sum();
    order.setPayAmount(order.calcPayAmount(order.getTotalAmount(), couponAmount));

    //检验账户余额是否足够
    if (order.getPayAmount() > 0L) {

        boolean balanceEnough = accountGateway.isBalanceEnough(placeOrderDto.getUserId(), order.getPayAmount());
        if(!balanceEnough) {
            throw new AppBusinessException(CommonErrorCode.BAD_REQUEST, "下单失败, 账户余额不足");
        }
    }

    //...

    //#4
    eventBus.ask(
            AskParameterBuilder.askOptional(askReduceBalance, askUseCoupon)
                    .callbackClass(OrderCreateCallback.class)
                    .addParam("orderId", String.valueOf(order.getId()))
                    .build()
    );

    return order;
}
```
代码略长, 我略过了其中一部分代码. `placeOrder`方法主要做的事情有:  
1.根据订购的产品ID, 向产品服务查询产品信息, 并计算订单金额.  
2.如果有使用优惠券, 向优惠券服务查询优惠券是否有效(请求REST接口).  
3.根据订单金额, 向账户服务查询用户余额是否足够(请求REST接口).  
4.如果上述步骤都成功完成, 发送账户扣款事件以及优惠券使用事件(如果有优惠券), 并注册回调方法等待事件结果.  
如果事件处理成功会调用`markCreateSuccess`方法, 处理失败会调用`markCreateFail`, `markCreateSuccess`方法的代码如下:
```java
@Transactional
public void markCreateSuccess(Long orderId) {
    Order order = checkOrderBeforeMarkSuccessOrFail(orderId);
    order.setStatus(OrderStatus.CREATED);

    orderRepository.save(order);
}
```
这个方法只是将订单状态置为`下单成功`, 流程就完成了. `markCreateFail`处理过程类似, 只不过是将订单状态改为`下单失败`.  

看到这里, 先不去管服务调用的实现细节, 细心的你可能会产生一些疑问:  
1.第4步为什么要使用事件的形式去扣款和处理优惠券, 不能和前面的查询操作一样使用REST接口来处理吗?  
2.为什么要先查询用户余额是否足够, 再发送扣款事件, 直接发送扣款事件不好吗? 如果查询余额返回成功之后, 其他业务修改了余额, 处理扣款事件的时候余额不足怎么办?  
3.第4步同时发送了扣款事件和优惠券使用事件, 如果扣款成功了, 但是优惠券使用失败了怎么处理?  

其实这些问题都指向同一个问题域: 分布式事务. 分布式事务是开发微服务首先要解决的问题.
分布式事务是一个很大的话题, 这里我只简单介绍一下eBay的Dan Pritchard提出的BASE原则:  
基本可用(Basically Available)  
软状态(Soft state)  
最终一致(Eventually consistent)  

BASE其实和传统数据库的ACID是两个不同的思想, 以我们上面的订单系统为例. 订单服务向账户服务发送扣款事件, 账户服务接收到事件并且处理成功, 但是还没有将处理结果发送到订单服务,
这时候系统数据就处于短暂的不一致状态: 用户的账户余额已经被扣减掉了, 但是订单状态还是`正在下单`. 过了一段时间, 订单服务获取到扣款事件的处理结果并且将订单状态置为`下单成功`.
这个时候系统才达到最终一致的状态. 这种事务处理方法并不是适用于所有业务, 如果需要强一致性, 还是得使用2PC或者3PC来完成.  

了解了mysteam的事务处理原则, 我们回头看看刚才提出的3个问题:  
1.mysteam是使用事件的方式来进行事务处理的, REST接口一般只用来实现查询或者其他不需要事务的操作. 所以只要涉及到数据修改, 一般都通过事件来完成.  
2.发送扣款之前先查询余额是为了减少不必要的事件操作, 因为如果事件处理失败会涉及到事件撤销, 是比较耗时的操作, 先进行余额查询, 余额不足直接流程就中止了.
根据我们的经验, 一般来说查询余额成功后续扣款失败的几率比较小, 所以收益大于付出.
3.这涉及到事件的撤销处理. 在mysteam的订单服务中, 如果接收到了扣款成功和优惠券使用失败这两个事件结果, 订单服务会启动事件撤销流程, 向账户服务发送扣款撤销事件, 并且将订单状态置为下单失败.  

综上, mysteam的事务处理遵循BASE, 实现方式是使用事件. 关于事务的其他细节以及事件如何实现, 我后面会用单独的文章来介绍.
这里我们先回到本篇的主题, 如何调用REST接口. 在这里, 我先简单介绍一下Eureka, Ribbon和Feign这三个组件.  
**[Eureka](https://github.com/Netflix/eureka)**: 服务注册中心. 我们的REST服务在启动的时候会将自己的地址注册到Eureka,
其他需要该服务的应用会请求Eureka进行服务寻址, 得到目标服务的ip地址之后就会使用该地址直连目标服务.  
**[Ribbon](https://github.com/Netflix/ribbon)**: 客户端负载均衡类库. 当客户端请求的目标服务存在多个实例时, Ribbon会将请求分散到各个实例. 一般会结合Eureka一起使用.  
**[Feign](https://github.com/OpenFeign/feign)**: HTTP客户端类库. 我们使用Feign提供的注解编写HTTP接口的客户端代码非常简单, 只需要声明一个Java接口加上少量注解就完成了.  

接下来我们看代码实例. 以账户服务的接口为例, 之前我们在`placeOrder`方法内查询账户余额的代码如下:
```java
boolean balanceEnough = accountGateway.isBalanceEnough(placeOrderDto.getUserId(), order.getPayAmount());
```
打开`$YOUR_PATH/mysteam/order/core/src/main/java/com/akkafun/order/service/gateway/AccountGateway.java`, 代码如下:
```java
@Service
public class AccountGateway {

    protected Logger logger = LoggerFactory.getLogger(AccountGateway.class);

    @Autowired
    AccountClient accountClient;

    @HystrixCommand(ignoreExceptions = RemoteCallException.class)
    public boolean isBalanceEnough(Long userId, Long amount) {
        return accountClient.checkEnoughBalance(userId, amount).isSuccess();
    }

}
```
AccountClient是一个加了Feign注解的接口:
```java
@FeignClient(AccountUrl.SERVICE_HOSTNAME)
public interface AccountClient {

    @RequestMapping(method = RequestMethod.GET, value = AccountUrl.CHECK_ENOUGH_BALANCE)
    BooleanWrapper checkEnoughBalance(@PathVariable("userId") Long userId, @RequestParam("balance") Long balance);

}
```
`@FeignClient`注解需要声明一个service id, 这个service id就是我们在YAML配置文件中配的`spring.application.name`的值, 比如`account.yml`中的`spring.application.name`值是`account`.
我们请求的REST接口需要一个url路径参数userId, 以及一个查询参数balance. 我们在代码中不需要直接调用Ribbon的代码, Feign会帮我们处理好一切.
根据我们的`AccountClient`接口声明, Feign会在Spring容器启动之后, 将生成的代理类注入`AccountGateway`,
所以我们不需要写HTTP调用的实现代码就能完成REST接口的调用.  

到这里下单的逻辑就完成了. 我们知道在分布式环境下, 服务之间的依赖都是脆弱而且不稳定的, 极有可能因为一个服务实例的延迟或宕机造成所有服务不可用.
所以mysteam中引入了hystrix. 细心的同学可能已经在`AccountGateway`中发现`@HystrixCommand`注解了, 下篇文章我将介绍hystrix的基本用法, 以及如何使用hystrix board和turbine来监控hystrix服务.

对这篇博客或者这个系列有问题的同学欢迎留言讨论: )
