---
layout:     post
title:      "Play Injection usage"
subtitle:   "play框架的依赖注入"
date:       2016-10-15 12:00:00
author:     "lanpay"
header-img: "img/post-bg-re-vs-ng2.jpg"
header-mask: 0.3
catalog:    true
tags:
    - scala
    - playframework
    - program
---

依赖注入的概念就不再赘述。从play24开始typesafe就鼓励使用依赖注入，同时它也移除了之前的global-setting模块。那么如何对依赖注入的部分进行初始化呢？



## Injection基本用法

```scala
"""
声明一个类MyComponent，并采用依赖注入的方式注入一个WSClient；
注意：在使用MyComponent时也需要用依赖注入方式构造！
"""
import javax.inject._
import play.api.libs.ws._
class MyComponent @Inject() (ws: WSClient) {
  // ...
}
``` 

## Singleton的使用

```scala
"""
一般采用注解方式声明单例！
"""
import javax.inject._
@Singleton
class MySingletonCls {
  @volatile private var price = 0
  def set(p: Int) = price=p
  def get = price
}
``` 

## Cleaning up

```scala
"""
当play shut down时，可能需要关闭一些资源，或处理善后。这时我们通过LifeCycle来做这件事情。
"""
import scala.concurrent.Future
import javax.inject._
import play.api.inject.ApplicationLifecycle

@Singleton
class MessageQueueConnection @Inject() (lifecycle: ApplicationLifecycle) {
  val connection = connectToMessageQueue()
  lifecycle.addStopHook { () =>
    Future.successful(connection.stop())
  }
  //...
}
``` 

## 依赖注入的初始化

```scala
"""
我们看到如果使用依赖注入，那么这一条线上的所有类都必须能够通过依赖注入生成。但是最顶层的类如何得到呢？一是可以将其注入到某个controller里面，这样play初始化controller时也会初始化这些类。二是使用bind进行绑定和初始化。
"""
class MyModule extends ScalaModule with AkkaGuiceSupport {
  def configure = {
    bind[TaxiMgr].asEagerSingleton() // singletong, 直接绑定
    bindActor[ScannerActor]("taxi-scanner-actor") // actor, 绑定时需要指定name
  }
}
``` 

***注意：由于akka的actor的ref是不带泛型的，因此只能通过绑定名字来区分。***


使用异步的特点就是，碰到原本需要blocking处理的操作就丢到下个线程池去处理。你可能会说，那不总归是得某个线程去处理么？确实如此。但是对比下同步操作吧，假设你有3个线程池完成一个任务的三个阶段，如果每个线程池都去同步等下个线程池的返回结果，那么一个任务会额外block住2个线程，而用异步呢？只有真正干活儿的那个线程被占用。





