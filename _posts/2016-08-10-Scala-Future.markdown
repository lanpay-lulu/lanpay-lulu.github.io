---
layout:     post
title:      "Scala Future usage"
subtitle:   "scala future 语句的用法总结"
date:       2016-08-10 12:00:00
author:     "lanpay"
header-img: "img/post-bg-re-vs-ng2.jpg"
header-mask: 0.3
catalog:    true
tags:
    - scala
    - future
    - program
---

如果你用过java的多线程或异步调用，一定对future不陌生。在scala中对future有了更完善的包装和使用方法，书写异步调用非常便捷。
future实质就是开一个线程，做指定计算，然后把结果返回。
相比thread，它有返回值，可以做回调，更加灵活。

## 基本用法

```scala
"""
创建一个future，其中调用了sleep模拟异步执行过程，结果返回1
注：Thread sleep 1000 等价于 Thread.sleep(1000)。Just a grammar sugar~ 
"""
val fu: Future[Int] = Future {
  Thread sleep 1000
  1
}
``` 

## 同步阻塞

```scala
"""
要想同步获得future的执行结果，可以用await，此时调用线程会阻塞在这里直到future返回或者超时。
"""
val result = Await result (fu, 3 seconds)
``` 

## 异步回调

```scala
"""
future的回调,我个人比较喜欢的一种写法~ ~
回调会在future结束后调用，如果成功会走map代码块，如果发生异常会走recover代码块。
"""
val callback = fu.map { x =>
  // do something
  println(s"callback! fu result = $x")
  0
}.recover{
  case t:Throwable =>
    t.printStackTrace()
    // handle exception
    -1
} 
``` 

***注意1：在future外围，你无法通过加try-catch来捕获future中的异常，此时应该使用recover方法。***


## 使用外围的context

```scala
"""
future可以使用外围的context变量，测试代码如下，会得到a1=1，a2=2的结果
"""
var a = 1
val fu1 = Future{
  println(s"a1=$a")
  Thread.sleep(1000)
  println(s"a2=$a")
}
Thread.sleep(500)
a = 2
``` 

***注意2：不要在future中使用return语句。***

## 多任务future

```scala
"""
多任务的异步调用方法如下。fuTask1和fuTask2会依次调用，这里要注意的是不能用condition guard，因为这里的for是被翻译成flatMap来执行的，同时也必须保证fuTask1和fuTask2必须是future类型返回值。
"""
val fus = for {
  res1 <- fuTask1()
  res2 <- fuTask2()
} yield (res1, res2)
fus.map{
  case (res1, res2) =>
    // do the rest with res1 and res2
}.recover{...}
```

```scala
"""
与之相似的一种更紧凑的写法如下
"""
(for {
  res1 <- fuTask1()
  res2 <- fuTask2()
} yield {
  // do anything with res1 and res2
}).recover{...}
```

注意：多任务的执行是顺序的，如果下一步执行需要依赖上一步的条件，可以使用predicate（参见上一篇[scala-for](https://lanpay-lulu.github.io/2016/07/24/Scala-For/)）。

## 嵌套future

```scala
"""
future的嵌套使用如下。fu2是一个Future[Future[Any]]类型，这并不方便我们在外围使用。fu3是使用flatMap将嵌套的future铺平了，它是一个Future[Any]类型，没有嵌套。
"""
val fu1 = fuTask1()
val fu2 = fu1.map{ x =>
  fuTask2(x)
}
val fu3 = fu1.flatMap{ x =>
  fuTask3(x)
}
```  

***注意3：flatMap内需要返回future类型。***


## future-list到list-future

我们在处理list或者seq时，可能会对其中每个元素都产生一个异步调用。此时我们希望能统一处理异步调用的回调。

```scala
"""
将一个future的list转化成list的future，方便拿到所有结果后再回调处理。
"""
val list = List()
val listFu = list.map { x =>
  fuTask(x)
}
Future.sequence(listFu).map { flist => 
  // do something
} 
``` 

## Execution Context 

最普遍的引入ec的写法

```scala
import scala.concurrent.ExecutionContext.Implicits.global
```

自己创建线程池

```scala
implicit val ec = ExecutionContext.fromExecutor(Executors.newFixedThreadPool(2))
```

显示指定线程池

```scala
"""
指定线程池。future中使用ec1，回调使用ec2.
"""
implicit val ec1 = ExecutionContext.fromExecutor(Executors.newFixedThreadPool(4))
implicit val ec2 = ExecutionContext.fromExecutor(Executors.newFixedThreadPool(4))
val fu = Future{/* do something */}(ec1)
fu.map{ x =>
  // do something
}(ec2)
``` 

***注意4：当不显示指定线程池时，默认用各自的implicit线程池执行。***


## Handle Exception

在处理future的exception时必须十分小心，因为外围的context是无法捕获future中的异常的，我们一般有两种方式来处理future的异常。一是使用recover，二是使用onComplete结合Success，Failure。

无论使用哪种方法，我们有如下几点经验。

```scala
Future{
  Future{
    throw new Exception("e1")
  }
  throw new Exception("e2")
}.map{ x =>
  // do something
}.recover{
  // can only capture e2
}

```

如果使用flatMap呢？

```scala
f1.flatMap{ x =>
  f2() // suppose f2 will throw exception e2
  f3().map{ xx => // suppose f3 will throw exception e3
    f4() // throw e4
  }
}.recover{
  // can only capture e3
  // can not capture e4
}

```

因此比较好的思路是对每个future都做容错处理！

## More about Future and Async

使用异步的特点就是，碰到原本需要blocking处理的操作就丢到下个线程池去处理。你可能会说，那不总归是得某个线程去处理么？确实如此。但是对比下同步操作吧，假设你有3个线程池完成一个任务的三个阶段，如果每个线程池都去同步等下个线程池的返回结果，那么一个任务会额外block住2个线程，而用异步呢？只有真正干活儿的那个线程被占用。





