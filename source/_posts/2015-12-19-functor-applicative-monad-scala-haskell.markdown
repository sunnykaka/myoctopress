---
layout: post
title: "简单介绍函数式编程中的Functor(函子)，Applicative(加强版函子)，Monad(单子)"
date: 2015-12-19 20:22:10 +0800
comments: true
categories: [函数式编程]
description: "简单介绍函数式编程中的Functor(函子)，Applicative(加强版函子)，Monad(单子)"
---

如果你是刚接触函数式编程，可能很容易被下面这些术语弄迷惑：Functor(函子)，Applicative(加强版函子)，Monad(单子)。
这些概念不是空穴来风，它们出自范畴论，如果你上网去搜范畴论，可能会找到大篇的术语定义，学术资料，这些资料大多都不是入门友好的。
这里我不会探讨定义，只会介绍这些概念在代码中到底起了什么样的作用，以及怎么样运用它们。

下面的示例代码大部分是Haskell，有一小部分是Java8，不会Haskell完全没关系，你可以把它们看作伪代码，我会对每一段代码进行解释。
这篇文章适合刚刚接触函数式编程的同学。我在刚接触这些概念的时候一头雾水，网上找的资料要么level太高看不懂，
要么直接就blabla给你介绍一大片背景知识了。后来经过长时间的摸爬滚打加实践，我发现这些概念理解起来也不是很困难，所以就想写一篇入门级的介绍。
如果你想要对函数式编程有一定的了解，这些概念你是绕不过去的，特别是Monad，当你发现你理解了Monad的机制，很多看起来不可思议的代码就能理解了。

开始之前，我简单介绍一下类型类(typeclass)和类型构造器的概念。函数式编程中的类型类是定义行为的接口。
如果一个类型是某个类型类的实例，那么这个类型必须实现所有该类型类所定义的行为。不要因为有“类”这个词就把类型类与面向对象中的类混淆，
他们是完全不同的概念。类型构造器能够接收其他类型为参数，创建出新的类型。举个例子，Scala的List即为接收一个类型参数的类型构造器，
当类型参数为Int时，List类型构造器的返回类型为List[Int]，当类型参数为String时，返回类型为List[String]。
与类型构造器相对的概念是值构造器，比如Int(2)。

### 1. Functor

首先看看函子的类型类用代码怎么表示：
```haskell
-- Haskell中一个函数如果有两个参数一个返回值，写法是这样: a(第一个参数) -> b(第二个参数) -> c(返回值)
class Functor f where
  fmap :: (a -> b) -> f a -> f b
```
函子的类型类只定义了一个fmap函数: fmap函数接收两个参数，第一个参数是以a为参数，b为返回值的函数；第二个参数类型为f a，fmap的返回值类型为f b.
注意这里的a, b可以为任意类型, f为接收一个类型参数的类型构造器。这样说可能有点抽象，来看一个具体的例子。
已知[]（列表）是一个functor实例，他的fmap函数声明为:
```haskell
fmap :: (a -> b) -> [a] -> [b]
```
接收一个以a为参数，b为返回值的函数以及元素类型为a的列表，返回元素类型为b的列表。
至此，你能看出functor所抽象的行为吗？你可以从下面两个角度思考fmap:
1. 接受函数和函子值，返回在函子值上映射函数的结果（返回也是函子值）。
2. 接受函数，把该函数从操作普通类型的函数**提升**(lift)为操作函子值的函数。
这就是函子，不难吧？
<!--more-->
### 2. Applicative
Applicative，俗称加强版函子，先来看Applicative类型类的代码:
```haskell
class (Functor f) => Applicative f where
  pure :: a -> f a
  (<*>) :: f (a -> b) -> f a -> f b
```
多了一个新的元素，先解释下。
`class (Functor f) => Applicative f`的意思是约束f类型必须首先是一个Functor(函子)，即如果一个类型是Applicative的实例，
则肯定是Functor的实例。Applicative类型类定义了两个函数：pure和`<*>`(其实还有一个fmap，
因为Applicative实例肯定是Functor的实例，所以fmap免费提供了)。pure是一个很简单的函数，接收任意类型的值为参数，返回包裹了该值的Applicative值。
`<*>`函数看起来和fmap有些像，唯一的区别是fmap的第一个参数接收一个普通函数(a -> b)，而`<*>`的第一个参数为f(a -> b)，
即把普通的函数用Applicative包裹。我们看看列表作为Applicative实例的实现:
```haskell
-- Haskell中，列表的写法为[], 例如[1,2,3]是一个元素都为Int的列表
pure x = [x]
(<*>) fs xs = [f x | f <- fs, x <- xs]
```
pure的实现很简单，把接收的参数值放入列表并返回。`<*>`的实现稍微复杂点，使用了列表生成式的语法。如果你接触过Python，对这种语法不会陌生，
这段代码如果用命令式语言风格翻译，会是这样:
```javascript
var array = [];
for(f in fs) {
  for(x in xs) {
    array.push(f(x));
  }
}
```
到此为止，我们应该已经了解Applicative实例的作用了，主要定义了两个行为，第一个行为是接收一个任意值为参数，返回一个函子值，
第二个行为是从一个函子值里取出函数，应用到第二个函子里面的值。那么Applicative有什么实际用处呢，看下面的应用：
```haskell
[(*0), (+100), (+200)] <*> [1, 2, 3]
-- 输出结果: 0,0,0,101,102,103,201,202,203
```
`(*0)`是对`*`函数的部分应用，`*`是一个二元函数，接收两个参数，返回这两个参数的乘积。`(*0)`是一个一元函数，接收一个参数，
返回这个参数与0的乘积。第一个列表里面是三个一元函数，分别应用到第二个列表的元素，参照我们之前对列表`<*>`函数的定义，很容易得出上面的结果。
这种把`<*>`串起来用的用法叫做Applicative风格，下面是另外几个例子：
```haskell
[(+), (*)] <*> [1, 2] <*> [3, 4]
-- 输出结果: [4,5,5,6,3,4,6,8]
pure (+) <*> [1, 2] <*> [3, 4]
-- 输出结果: [4,5,5,6]
```
非常好，现在我们能够把应用到普通值的函数应用到函子值上面了。

### 3. Monad
相比Functor和Applicative，Monad的应用更加广泛，Monad可以看作加强版的Applicative，引用<<Haskell趣学指南>>中的句子：
```
monad是对applicative函子概念的延伸，它们提供了下面这个问题的一种解决方案：如果我们有一个带有上下文的值m a，如何对它应用这样一个函数——取
类型为a的参数，返回带上下文的值？换句话说，怎么对m a应用类型为a -> m b的函数？
```
这种场景非常常见，我们来看Java8的Stream API:
```java
public interface Stream<T> extends BaseStream<T, Stream<T>> {
  //...
  <R> Stream<R> flatMap(Function<? super T, ? extends Stream<? extends R>> mapper);
  //...
}
```
flatMap方法接收一个函数（这个函数接收T类型的入参，返回Stream<R>）然后在Stream<T>上应用这个函数，返回Stream<R>(暂时不考虑子类问题)。
我们来看看Haskell中的"flatMap"（Haskell中叫做绑定）:
```haskell
(>>=) :: (Monad m) => m a -> (a -> m b) -> m b
```
`>>=`函数接收两个参数：一个Monad值m a，一个函数（入参为类型a，返回值为Monad值m b），返回值类型为Monad值m b。
这和Java8的flatMap形式上非常相似，其实它们要解决的是相同的问题。
Monad在函数式编程中无处不在，来看看具体的例子。
```haskell
[3,4,5] >>= \x -> [-x]
-- 输出结果: [-3,-4,-5]
```
首先，列表是一个Monad，对照`>>=`函数的定义，这里的m就是列表，a就是Int，a -> m b对应的类型就是Int -> [Int]。
上面的逻辑用Java8的flatMap实现：
```java
//Lists来自Guava库
Lists.newArrayList(3,4,5).stream().flatMap(x -> Lists.newArrayList(-x).stream()).collect(Collectors.toList());
```

好了，这里我简单介绍了Functor，Applicative，Monad的概念以及实际应用。如果对这些概念想更进一步了解，可以看看下面的书：  
1. [Haskell趣学指南](http://book.douban.com/subject/25803388/)  
2. [Functional Programming in Scala](http://book.douban.com/subject/20488750/)  
3. [范畴论](http://book.douban.com/subject/1894611/)  
