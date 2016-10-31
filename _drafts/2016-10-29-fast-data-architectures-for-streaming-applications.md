---
layout: default
title: fast data architectures for streaming applications
---

此文为同名书籍[fast data architectures for streaming applications]("https://info.lightbend.com/COLL-20XX-Fast-Data-Architectures-for-Streaming-Apps_LP.html?lst=WS&_ga=1.181793319.1638701083.1472450513")
的一些阅读感想和总结。

## 大数据简史

writting sth about akka-stream

##Introduction
motivation
我们假设现在从互联网上获取的服务包含很多流式数据处理的实例，包括从一个服务中进行下载和上传，或者点到点（p2p）数据传输。把数据当作一个包含很多元素的流，而不是整体中的一个
非常有用，因为这种想法符合电脑发送和收到数据的方式（例如通过tcp），但是它通常也是一种必要条件，因为数据集通常变得很大而不能作为一个整体被处理。我们分开计算或者鸡西通过
一个大集群，并称之为“大数据”，处理它们的总体准则是将这些数据按顺序的-作为一个流-通过一些cpu处理。
actors看起来能够一样优秀的处理流式数据。它们按顺序发送和接收一系列的消息为了将知识（或数据）从一个地方传到到另外一个地方。我们发现它乏味和容易出错去按顺序实现所有正确的操作。
在actor之间去获取稳定的流。因为为了发送和接收我们也需要注意不要溢出内存或者进程中的mailbox。另外一个陷阱是actor消息会丢失，并一定会重传，在这种情况下，免得流在获取数据端
有坑。当处理有很多被给定类型的元素数据流时。actors现在也不能提供一个很好的稳定的保证没有连线错误会被生成，这种情况下类型安全能够提升。
由于这些原因我们决定对这些问题打包一个解决方案作为akka-stream-api。这的目的是为了提供一个直观和安全的方式去制定流式操作处理建立，这样我们能接着能有效率的处理数据，同时
包含了资源的使用，再也不会有内存溢出的错误。为了达到这个目的，我们的流需要能够限制它们使用的内存，它们需要能够降低生产者的速率如果消费者不能跟上节奏。这个特性被称作
背压(back-pressure)，这也是响应式流(akka是基础成员)的核心倡议。对于你，这意味着传播和对于背压的响应难题已经解决在了AKKA-STREAM的设计中，所以你可以少考虑一件事情。这也
意味着akka-stream可以和其他响应式流的实现（响应式流的接口定义了互操作SPI,例如akka-stream这样的实现提供了一个优雅的使用者API）进行无缝互操作。
relationship with reactive streams
AKKA-Stream的API完全和响应式流的接口解耦。Akka

