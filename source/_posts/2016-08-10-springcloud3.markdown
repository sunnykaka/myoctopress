---
layout: post
title: "微服务框架Spring Cloud介绍 Part3: mysteam项目结构与开发用户注册服务"
date: 2016-08-10 10:06:10 +0800
comments: true
categories: [微服务, Spring Cloud]
description: "微服务框架Spring Cloud介绍 Part3: mysteam项目结构与开发用户注册服务"
---

在[上一篇文章](http://skaka.me/blog/2016/08/04/springcloud2/)中我们简单的了解了一下Spring Cloud.
因为Spring Cloud相关的内容较多, 所以我建了一个项目mysteam来演示Spring Cloud的使用, [GitHub地址](https://github.com/sunnykaka/mysteam).

这是一个Maven项目, 下载下来之后直接导入IDE, 你会看到如下的项目结构(我用的是Intellij IDEA):
{% img /images/custom/20160810/mysteam_structure.png %}

普通目录:  
docs: 存放文档资料, 例如数据库脚本, astah文件(UML工具)等.  
logs: 运行日志存放目录.  
公共模块:  
apiutils: api模块公共父模块.  
common: 服务模块公共父模块, 存放微服务共同依赖的逻辑, 例如事件处理, 定时任务等.  
utils: 工具类模块.  
基础服务模块:  
eureka: eureka服务. 提供服务注册与服务发现. 这个服务之后会有专门的文章来介绍.  
config: config服务. 提供配置管理服务. 这个服务之后会有专门的文章来介绍.  
turbine: hystrix服务监控. 这个服务之后会有专门的文章来介绍.  
服务模块:  
account: 账户服务.  
coupon: 优惠券服务.  
order: 订单服务.  
product: 产品服务.  
user: 用户服务.  
其他模块:  
integration-test: 集成测试模块.  

这些模块内部的项目结构大多类似, 以服务模块user为例.  
api: api接口模块. 其他依赖user服务的服务会依赖这个模块.  
core: user服务实现模块.  
api和core模块内容都是标准的maven项目结构, 其中core模块主要有这么一些子目录:  
context: 存放Spring Boot启动类.  
dao: DAO层.  
domain: Model层.
service: Service层.  
web: 存放Spring MVC Controller.  

值得特别说明的是, 在真实的项目中, 一般每个服务都是一个独立的项目, 彼此之间只是通过pom引用. 如果代码都放到一个项目中,
过一段时间你会发现每次打开IDE都是件痛苦的事情, 而且IDE运行速度会奇慢无比. 这样做也违背了微服务开发的本意: 各个服务之间相对独立.
mysteam把所有的服务都放到一个项目中只是为了方便演示和运行. 如果你想将mysteam的模块都拆到独立项目中去也是相当的简单, 只要修改pom文件即可.  

好了, 项目结构介绍完, 接下来我们要做点正事了: ) 实现用户注册服务.
用户表的结构相当简单, 只有三个字段. sql文件在`$YOUR_PATH/mysteam/user/docs/user-service.sql`. 我们首先创建实体类.
文件位置在`$YOUR_PATH/mysteam/user/core/src/main/java/com/akkafun/user/domain/User.java`.
```java
@Entity
@Table(name = "user")
public class User extends VersionEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.AUTO)
    private Long id;

    @Column
    private String username;

    @Column
    private String password;

    public Long getId() {
        return id;
    }

    public void setId(Long id) {
        this.id = id;
    }

    public String getUsername() {
        return username;
    }

    public void setUsername(String username) {
        this.username = username;
    }

    public String getPassword() {
        return password;
    }

    public void setPassword(String password) {
        this.password = password;
    }


}
```
实体类很简单, 使用的是JPA注解, 继承抽象基类VersionEntity来获得乐观锁控制功能.  

DAO层使用的是[Spring Data JPA](http://docs.spring.io/spring-data/data-jpa/docs/current/reference/html/),  
目录在`$YOUR_PATH/mysteam/user/core/src/main/java/com/akkafun/user/dao`, DAO相对简单也不是重点, 这里就不介绍了.  
如果不喜欢JPA, 可以很容易地替换成MyBatis.

Service类是`$YOUR_PATH/mysteam/user/core/src/main/java/com/akkafun/user/service/UserService.java`, 我们看一下用户注册的业务逻辑:
```java
@Transactional
public User register(RegisterDto registerDto) {
    if(isUsernameExist(registerDto.getUsername(), Optional.empty())) {                         //1
        throw new AppBusinessException(UserErrorCode.UsernameExist,
                String.format("用户名%s已存在", registerDto.getUsername()));
    }

    User user = new User();
    user.setUsername(registerDto.getUsername());
    try {
        user.setPassword(PasswordHash.createHash(registerDto.getPassword()));
    } catch (GeneralSecurityException e) {
        logger.error("创建哈希密码的时候发生错误", e);
        throw new AppBusinessException("用户注册失败");
    }

    userRepository.save(user);                                                                  //2

    //用户创建事件
    eventBus.publish(new UserCreated(user.getId(), user.getUsername(), user.getCreateTime()));  //3

    return user;
}

@Transactional(readOnly = true)
public boolean isUsernameExist(String username, Optional<Integer> userId) {
    return userRepository.isUsernameExist(username, userId);
}

```
1. 注册之前首先判断用户名是否存在, 判断逻辑在UserRepositoryImpl类里. 如果用户名重复就抛出异常.  
2. 调用DAO的save方法持久化用户到数据库.  
3. 发送用户创建事件.  

注意register方法上有`@Transactional`注解, 代表事务边界是在service层. register方法构成一个事务, 包括事件发送.
关于事件处理后续有专门的文章描述, 这里先略过.

现在来看下Controller层的处理. 打开`$YOUR_PATH/mysteam/user/core/src/main/java/com/akkafun/user/web/UserController.java`:
```java
@RestController
@RequestMapping(produces = MediaType.APPLICATION_JSON_VALUE)                  //1
public class UserController {

    @Autowired
    UserService userService;

    @RequestMapping(value = USER_REGISTER_URL, method = RequestMethod.POST)
    public UserDto register(@Valid @RequestBody RegisterDto registerDto) {    //2

        User user = userService.register(registerDto);                        //3
        UserDto userDto = new UserDto();
        userDto.setId(user.getId());
        userDto.setUsername(user.getUsername());

        return userDto;
    }


}
```
这就是一个很普通的Spring MVC Controller, 有几个地方需要注意一下.
1. 我们的Rest服务暂且只提供json数据的请求和响应, 所以在class级别加了一个注解`@RequestMapping(produces = MediaType.APPLICATION_JSON_VALUE)`.
2. 注册是POST请求, 我们使用DTO对象RegisterDto来收集数据. 注意RegisterDto是user服务的api模块提供的, 意味着其他依赖了user服务的模块可以直接使用RegisterDto.
RequestBody类使用了[Java Validation](http://docs.spring.io/spring/docs/current/spring-framework-reference/html/validation.html)注解来校验参数的合法性.
3. 调用UserService的register方法完成注册, 然后将User实体对象转化成UserDto对象返回.

到此就开发完了. 现在我们可以启动user服务来看一下效果.
**提示: 运行下面的UserApplication之前, 需要先启动Eureka服务和Config服务, 启动方法请参考[上一篇文章](http://skaka.me/blog/2016/08/04/springcloud2/).**
打开`$YOUR_PATH/mysteam/user/core/src/main/java/com/akkafun/context/web/UserApplication.java`, 直接运行main方法.
项目启动之后, 在浏览器访问http://localhost:23101/swagger-ui.html, 你应该能看见如下的页面:
{% img /images/custom/20160810/user_swagger_ui.png %}

这个页面是[SpringFox](http://springfox.github.io/springfox)根据我们的Controller类, 自动生成的swagger ui页面.
关于swagger和SpringFox, 之后会有专门的文章来介绍. 这个页面列出了user服务下所有的api信息(暂时只有一个register), 包括url链接, 请求参数, 返回值等,
你也可以在Controller类中加入`@ApiOperation`这种Swagger注解来对接口进行更详细的描述. 此外, 在这个页面你还可以直接对api进行测试, 例如在registerDto参数栏填入
```
{
  "password": "123456",
  "username": "aaa"
}
```
然后点击下面的Try it out!按钮, 你就能看见服务器的返回结果了.  

大功告成. 整个过程除去实体类的话, 真正的业务代码只有几十行. 代码量虽少, 但是我们已经开发了一个完整的注册服务, 用户注册服务不但自动生成了完整的API文档,
同时已经能通过Eureka被其他服务调用了(下一篇文章演示). 当然, 这一切都仰仗于Spring Cloud, Netflix OSS, SpringFox, Swagger等一系列开源软件的帮助, 程序员的生产力也因此越来越高.  
看看上面的步骤, 你也许会觉得, 开发一个微服务也是相当简单的嘛. 事实上, 我们还没有接触到真正的难点, 因为服务之间还没有交互.
下篇文章我会通过下单服务, 介绍如何进行服务之间的相互调用以及如何处理事件来保证事务完整性.
