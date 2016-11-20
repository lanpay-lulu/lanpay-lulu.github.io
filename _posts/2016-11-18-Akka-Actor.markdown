---
layout:     post
title:      "Akka actor"
subtitle:   "Akka-actor简介"
date:       2016-11-18 12:00:00
author:     "lanpay"
header-img: "img/post-bg-re-vs-ng2.jpg"
header-mask: 0.3
catalog:    true
tags:
    - akka
    - actor
    - engineer
---

### Actor简述

Akka actor主要用来处理高并发的分布式事务。它实际上是一种处理事物的思想，即actor通过消息来传递信息，每个actor顺序地处理收到的消息，并且不保证消息一定收到或者一定处理好。在很大程度上我们用actor来解决同步问题，比如把相同的msg丢给同一个actor，这样可以有效避免竞争。

### 典型使用示例

```scala
import akka.actor.Props
import akka.actor.ActorRef
import akka.actor.{Actor, ActorRef, Props}
import scala.util.Random

sealed trait MyMessage // sealed接口仅能在当前文件被继承
case class HelloMsg(msg:String, uid:Int) extends MyMessage

object MyActorTest {
  def main(args : Array[String]) : Unit = {
    val system = ActorSystem("MyActorSystem")
    myActor = system.actorOf(Props[MyActorDispatcher], name = "myActor")
    val uid = Random.nextInt()
    val callable = system.scheduler.schedule(
      0.seconds, 4.seconds, myActor, HelloMsg("hello", uid))  // schedule用来延迟发送消息或循环发送消息
  }
}

class MyActorDispatcher extends Actor {
  var workerList: List[ActorRef] = List()
  initWorkerList()

  def initWorkerList(): Unit = {
    for (i <- 1 to updateActorNum) {
      val act = context.actorOf(Props[MyWorker], name = "UpdateWorker" + i)
      workerList = act :: workerList
    }
  }

  def routing(key: Int): ActorRef = {
    val idx = workerList.length % key
    workerList(idx)
  }

  def receive = {
    case HelloMsg(msg, uid) =>
      val worker = routing(uid)
      worker ! HelloMsg(msg, uid)
    case _ =>
      println("error!")
  }
}

class MyWorker extends Actor {
  def receive = {
    case HelloMsg(msg, uid) =>
      println(s"Hello! msg=$msg, uid=$uid")
    case _ =>
      println("error!")
  }
}
```

其中包括了：
- 如何自定义做routing；通过自定义，我们可以把某类msg发给指定actor处理，解决它们之间的race condition；
- 如何延迟发送msg或循环发送msg；循环发送时需要注意，是按固定间隔发送，还是在actor处理完一条消息以后再延迟发送；


### Actor with response

以上的示例中，外部代码无法获取actor的执行结果。那么如果我们需要获取执行结果应该如何写呢？

```scala
  """ 通过mapTo函数能够获得actor的返回结果 """
  val fu: Future[Int] = (actorRef ? msg).mapTo[Int]
```

我们看到，除了 “!”，我们还可以通过 “?” 来向actor发送消息，它们的区别如下：
- ！（等价于tell函数）发送消息给actor是不会得到应答的；
- ？（等价于ask）发送消息给actor会获取应答；
- forward 发送消息类似ask，区别在于，A发给B然后B forward给C，C会直接应答给A，即B只是起一个转发的作用；


### 进阶

有了以上两个示例，我们已经可以用actor来做一些事情，但是我们在使用过程中还会发现一些问题，尤其是我们使用actor来处理事务的时候。比如在一个订单系统中，我们需要对某个订单做操作，如：支付、取消等，那么我们需要确保对它的操作是顺序的，否则同时对一个订单执行多个操作就会造成竞争。这时我们可以使用actor来做，对同一个订单的操作丢给同一个actor，那么自然能保证操作是顺序执行的。

这里面会遇到的问题是，actor会调用一些异步函数，如访问网络、数据库等；那么只有在结果返回来以后该次操作才算完成。但是在scala中我们尽量避免使用await操作，它会造成多个线程阻塞。好了，问题来了

```scala
  """ 通过mapTo函数能够获得actor的返回结果 """ 
  actorRef ? payMsg
  actorRef ? cancelMsg
  // ...

  def receive = {
    case PayMsg(orderId) =>
      val fu = visitDB().map{ ... }
    case CancelMsg(orderId) =>  
      val fu = visitDB().map{ ... }
```

看如下示例，receive函数的结果在future回调以后才知道，但是future尚未回调时，如果不使用await的同步操作，函数就会结束，意味着actor可以开始处理下一个请求。这样它就没有保证事务的按顺序执行。

那么我们如何避免这种情况呢？好在akka-actor提供了stash功能，能够将actor挂起，在挂起状态下不处理新的msg直到恢复正常状态。这样我们就能在future返回前将其挂起，在future返回后将其恢复正常。示例代码如下：

配置actor的mailbox类型：

```scala
akka {
  actor {
    my-custom-dispatcher {
      mailbox-type = "akka.dispatch.UnboundedDequeBasedMailbox"
    }
  }
}
```

actor示例：

```scala
sealed trait MyMessage // sealed接口仅能在当前文件被继承
case class HelloMsg(msg:String, uid:Int) extends MyMessage
case class ResumeMsg() extends MyMessage

class MyActor extends Actor with Stash{
  implicit val ec1 = MyTheadPools.ec1
  val ec2 = MyTheadPools.ec2
  def receive = {
    case HelloMsg(msg, uid) =>
      // 让actor状态变成挂起消息, 要传入一个函数来做消息挂起操作
      context.become(waiting, discardOld = false)
      handleWork(uid).onComplete{
        case Failure(e) =>
          println(s"An error occurred! e=$e")
          e.printStackTrace()
          self ! ResumeMsg
        case Success(v) =>
          println(s"handle result = $v")
          self ! ResumeMsg
      }
    case _ =>
      println("Error Msg received!")
  }

  def waiting: Receive = {
    case ResumeMsg =>
      context.unbecome() // 结束挂起状态，恢复正常处理数据
      unstashAll() // 将挂起消息放入队列处理
    case _ =>
      stash() // 将消息挂机
  }

  def handleWork(uid: Int): Future[Int] = {
    Future{
      val time = Random.nextInt(200)
      println(s"sleep time = $time")
      Thread.sleep(time)
      uid + 1
    }(ec2)
  }
}
```

通过一个resume消息，我们将actor状体恢复正常


## 其它

### 注意事项

- Actor返回的都是ActorRef，它没有泛型，使用时应该注意这点；
- 不要在Actor中block住线程，例如使用Await等操作，这很容易让整个线程池阻塞住；
- 获取发送方的sender()方法，需要在actor的receive的执行线程内获取，而不能在异步调用的回调中获取；
- 不同actor之间不应该共享状态，总是使用消息来发送状态；


### 分布式actor

To be continued ...



