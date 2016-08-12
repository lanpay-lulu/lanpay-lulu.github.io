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

** 注意1：在future外围，你无法通过加try-catch来捕获future中的异常，此时应该使用recover方法！**


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

** 注意2：不要在future中使用return语句。 **

## 多任务future

```scala
"""
多任务的异步调用方法如下。fuTask1和fuTask2会依次调用，这里要注意的是不能用condition guard，因为这里的for是被翻译成flatMap来执行的。
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

## future-list 和 list-future

我们在处理list或者seq时，可能会对其中每个元素都产生一个异步调用。此时我们希望能统一处理异步调用的回调。

```scala
"""

"""
val list = List()
val listFu = list.map { x =>
  fuTask(x)
}
Future.sequence(listFu).map { flist => 
  // flist是调用返回结果的list
} 
``` 

## Execution Context 

最普遍的引入ec的写法
`import scala.concurrent.ExecutionContext.Implicits.global
`

自己创建线程池
`  implicit val ec1 = ExecutionContext.fromExecutor(Executors.newFixedThreadPool(2))
`

** 注意3：执行future的线程池可以指定，执行future结束后回调的线程池同样可以指定。默认是用各自的原线程池执行。 **



```scala
"""

"""

``` 


并在recover中被捕获。为什么呢？因为这时for被翻译成flatMap语句了(见[这里](http://docs.scala-lang.org/tutorials/FAQ/yield.html))，而不是常规上的for语句。

那么在for中如何处理这种future呢？

```scala
def predicate(condition: Boolean)(fail: Exception): Future[Unit] = 
  if (condition) Future( () ) else Future.failed(fail)
def test1(n: Int)
  val fu = for {
    a <- ft1(n)
    _ <- predicate( a != 0 )(new MyException("divide by zero"))
    b <- ft2(n)  // ft2 will only run if the predicate is true
} yield (a, b)
```

然后同样是在recover中将其捕获并处理。你可能会问，这和之前的写法没什么区别嘛。这么写的好处在于你可以自定义Exception并根据异常做相应处理，而不是等到值被传入后面的future了再抛出来。



