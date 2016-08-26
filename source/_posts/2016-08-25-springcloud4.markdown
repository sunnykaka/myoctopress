---
layout: post
title: "微服务框架Spring Cloud介绍 Part4: 使用REST"
date: 2016-08-10 10:06:10 +0800
comments: true
categories: [微服务, Spring Cloud]
description: "微服务框架Spring Cloud介绍 Part3: mysteam项目结构与开发用户注册服务"
---

在[上一篇文章](http://skaka.me/blog/2016/08/10/springcloud3/)中我们开发了一个用户注册服务. 这篇文章我将介绍如何开发mysteam订单服务中的下单功能,
下单功能会涉及服务之间的交互与事件的处理, 并且我会对开发过程中用到的框架和类库进行简单地讲解. 开始写代码之前, 我们先来看看下单的处理流程:
{% img /images/custom/20160825/place_order.png %}

注意, 1,2,3,4,11步的黑色箭头代表是同步操作, 5,6,7,8,9,10步是异步操作.
下单接口接收要订购的产品ID, 数量和要使用的优惠券ID, 然后调用产品服务的接口查询产品信息, 调用优惠券接口校验优惠券是否有效,
以及调用账户接口判断账户金额是否足够(mysteam是一个虚拟物品商城, 采用先充值后购买的形式). 如果这些校验都成功, 订单服务会发送账户扣款事件和优惠券使用事件到MQ,
账户服务和优惠券服务会从MQ读取事件进行处理, 如果处理成功, 订单服务将能接收到结果, 并且将订单状态置为`下单成功`, 如果处理失败或超时, 订单状态会被置为`下单失败`.  

整个处理流程简单来说就是这样. 当然中间省略了很多细节, 比如怎么保证扣款和优惠券状态修改的原子性, 以及事务怎么处理超时, 超时后又怎么进行撤销和补偿等等. 这些内容是个大坑,
但是和Spring Cloud的主题相关性并不是太大, 所以我打算把事务有关的内容放在系列的最后进行介绍. 现在涉及事务的地方我都会简单带过, 这不会影响到对Spring Cloud的理解.

####1. 实现Model
流程清楚了, 现在我们来动手实现吧. 实体类的代码我就不贴了, 订单类是`com.akkafun.order.domain.Order`, 订单类和前面的用户类类似, 订单类中有两个字段需要注意一下:
```java
@Column
@Enumerated(value = EnumType.STRING)
private OrderStatus status;

@OneToMany(fetch = FetchType.LAZY, mappedBy = "order")
private List<OrderItem> orderItemList = new ArrayList<>(0);
```
OrderStatus是订单状态, 是一个枚举. 而OrderItem就是订单项.  

####2. 实现DAO
DAO层基本没有实际的代码, 就不贴了.  

####3. 实现Service
打开`$YOUR_PATH/mysteam/order/core/src/main/java/com/akkafun/order/service/OrderService.java`, 下单的逻辑在placeOrder方法内:
```java
/**
 * 下订单
 *
 * @param placeOrderDto
 * @return
 */
@Transactional
public Order placeOrder(PlaceOrderDto placeOrderDto) {
    Order order = new Order();
    order.setUserId(placeOrderDto.getUserId());
    order.setStatus(OrderStatus.CREATE_PENDING);
    order.setOrderNo(Long.parseLong(RandomStringUtils.randomNumeric(8)));

    //查询产品信息
    List<Long> productIds = placeOrderDto.getPlaceOrderItemList().stream()
            .map(PlaceOrderItemDto::getProductId)
            .collect(Collectors.toList());

    //#1
    List<ProductDto> productDtoList = productGateway.findProducts(productIds);
    Map<Long, ProductDto> productDtoMap = productDtoList.stream()
            .collect(Collectors.toMap(ProductDto::getId, Function.identity()));

    List<OrderItem> orderItemList = placeOrderDto.getPlaceOrderItemList().stream().map(placeOrderItemDto -> {
        OrderItem orderItem = new OrderItem();
        orderItem.setProductId(placeOrderItemDto.getProductId());
        orderItem.setQuantity(placeOrderItemDto.getQuantity());
        ProductDto productDto = productDtoMap.get(placeOrderItemDto.getProductId());
        orderItem.setPrice(productDto.getPrice());
        return orderItem;
    }).collect(Collectors.toList());
    order.setOrderItemList(orderItemList);

    order.setTotalAmount(order.calcTotalAmount());

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

    //计算订单金额
    long couponAmount = orderCouponList.stream().mapToLong(OrderCoupon::getCouponAmount).sum();
    order.setPayAmount(order.calcPayAmount(order.getTotalAmount(), couponAmount));

    //检验账户余额是否足够
    if (order.getPayAmount() > 0L) {

        //#3
        boolean balanceEnough = accountGateway.isBalanceEnough(placeOrderDto.getUserId(), order.getPayAmount());
        if(!balanceEnough) {
            throw new AppBusinessException(CommonErrorCode.BAD_REQUEST, "下单失败, 账户余额不足");
        }
    }

    //保存订单信息
    orderRepository.save(order);
    orderItemList.forEach(orderItem -> {
        orderItem.setOrderId(order.getId());
        orderItemRepository.save(orderItem);
    });
    orderCouponList.forEach(orderCoupon -> {
        orderCoupon.setOrderId(order.getId());
        orderCouponRepository.save(orderCoupon);
    });

    //解决订单金额为0还发送请求的问题
    Optional<AskReduceBalance> askReduceBalance = Optional.empty();
    if(order.getPayAmount() > 0L) {
        askReduceBalance = Optional.of(new AskReduceBalance(placeOrderDto.getUserId(), order.getPayAmount()));
    }
    Optional<AskUseCoupon> askUseCoupon = Optional.empty();
    if(!orderCouponList.isEmpty()) {
        List<Long> couponIds = orderCouponList.stream().map(OrderCoupon::getCouponId).collect(Collectors.toList());
        askUseCoupon = Optional.of(new AskUseCoupon(couponIds, placeOrderDto.getUserId(), order.getId()));
    }
    if(!askReduceBalance.isPresent() && !askUseCoupon.isPresent()) {
        markCreateSuccess(order.getId());

    } else {
        eventBus.ask(
                AskParameterBuilder.askOptional(askReduceBalance, askUseCoupon)
                        .callbackClass(OrderCreateCallback.class)
                        .addParam("orderId", String.valueOf(order.getId()))
                        .build()
        );
    }

    return order;
}
```
1.调用productGateway查询产品信息.  
2.
