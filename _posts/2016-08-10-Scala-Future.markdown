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


```scala
"""
要想同步获得future的执行结果，可以用await，此时调用线程会阻塞在这里直到future返回或者超时。
"""
val result = Await result (fu, 3 seconds)
``` 

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

** 注意：在future外围，你无法通过加try-catch来捕获future中的异常，此时应该使用recover方法！！！**


### 遍历集合
遍历seq

```scala
val seq = Seq(1,2,3,4,5)
for(i <- seq) { 
  println(s"$i")
}
```

遍历map

```scala
for((k,v) <- map) {
  println(s"key=$k, val=$v")
}
for(k <- map.keys) {
  println(s"key=$k, val=${map.get(k)}")
}
```

## 进阶用法

### 嵌套循环
我们知道一般语言如c++，2层嵌套循环需要写2个for语句如下：

```cpp
for(int i=0; i<5; i++)
  for(int j=i; j<10; j++)
     printf("%d, %d\n", i, j)
``` 

在scala中可以优雅的写出嵌套循环：

```scala
for(i <- 0 to 4; j <- i to 10)
  println(s"i=$i, j=$j")
```

### 条件语句（guard）
scala的for循环可以做if做条件检查，其中的if语句称为guard。

```scala
// multi-guard
val map = Map(1 -> "1", 2 -> "2", 3 -> "3")
for((k,v) <- map if v < "3"; if k != 1)
  println(s"key=$k, val=$v")
```

### yield
for ... yield ... 语句将根据for中的子句迭代产生出一个yield子语句构成一个集合，集合类型是根据for的子句推导出的。

```scala
val res = for(i <- 1 to 5) yield i
println(s"res = $res")
// 生成的是vector
for (c <- "Hello"; i <- 0 to 1) yield (c + i).toChar
// 将生成 "HIeflmlmop"，string类型
```

## 高阶用法
for语句除了用在循环中，还经常配合future语句使用。

### for And future
```scala
def ft1(n: Int) = Future(n+2)
def ft2(n: Int) = Future(24/n)
def test1(n: Int) = {
  for{
    a <- ft1(n)
    b <- ft2(n)
  } yield (a, b)
}
```

如果你使用过future，就知道future是异步调用，当我希望回调函数同时拿到几个异步调用的返回结果时，就可以使用for语句。
注意：处理future时，不能使用guard语句。

### for，future，guard And exception
```scala
def ft1(n: Int) = Future(n+2)
def ft2(n: Int) = Future(24/n)
def test1(n: Int) = {
  val fu = for{
    a <- ft1(n)
    b <- ft2(n) if n>0 // guard will not working
  } yield (a, b)
  fu.map{
    case (a,b) =>
      println(s"a=$a, b=$b")
  }.recover{
    case e: Throwable =>
      e.printStackTrace()
  }
}
```

对于ft2函数，传入0会抛出异常，即是你加上guard，该异常仍然会发生，并在recover中被捕获。为什么呢？因为这时for被翻译成flatMap语句了(见[这里](http://docs.scala-lang.org/tutorials/FAQ/yield.html))，而不是常规上的for语句。

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



