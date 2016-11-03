---
layout: default
title: akka stream learning
---

## reactive tweets

对于流进行操作的一个经典用法是消费一个我们想从中提取或合并数据的的流.在这个例子中，我们会考虑消费tweets的流，然后着重于用AKKA
从中提取信息。
我们也会考虑到所有无阻塞流解决方案的固有问题：如果订阅者太慢而无法正常消费数据。通常这种解决方案是把这些元素放入内存，但是这样会
通常会最终引起的内存溢出和这类系统的不稳定性。AKKA STREAM相反，依赖于内部的背压信号这样就允许控制在这样的场景下应该发生什么。
下面是我们将要工作的数据模型，通过快速开始idea例子看：

    final case class Author(handle: String)
     
    final case class Hashtag(name: String)
     
    final case class Tweet(author: Author, timestamp: Long, body: String) {
      def hashtags: Set[Hashtag] =
        body.split(" ").collect { case t if t.startsWith("#") => Hashtag(t) }.toSet
    }
     
    val akkaTag = Hashtag("#akka")
    
## transforming and consuming simple streams    
    
通过这个应用示例，我们看到的是一个普通的Twitter输入流，我们希望从中获取需要的信息，例如找到所有发tweet已#akka开头的用户。
为了准备我们的环境通过创建一个actor-sys和actor-materializer，这些是我们将要创造的物化和运行streams所需要的：
    
    implicit val system = ActorSystem("reactive-tweets")
    implicit val materializer = ActorMaterializer()
    
这个ActorMaterializer能够接收一个setting作为参数（能被用来定义materializer的属性），例如默认的内存大小，这个分发器也被用来
管线化pipeline,这些能被重写使用withAttributes方法，在Flow,Source,Sink和Graph上。
假设我们有一个tweets流已经可用了，在AKKA中被表示为*Source[Out,M]*

    val tweets: Source[Tweet, NotUsed]
    
流通常开始流动从Source开始，然后接着流过Flow,或者更多更高级的graph元素，最后被sink消费掉。
这些操作看起来和任何一个使用scala集合库一样，然而它们操作于流之上，而不是一个数据集。（这是非常重要的区别，因为一些操作只在流上有意义，反之亦然）
    
    val authors: Source[Author, NotUsed] =
      tweets
        .filter(_.hashtags.contains(akkaTag))
        .map(_.author)

最后为了物化和运行流计算，我们需要绑定flow到Sink上是的flow得以运行。最简单的做法是在一个source上使用callWith（sink）方法。为了简便，
一系列普通的sinks被预先定义了，然后收集起来像方法一样放在了sink的伴生对象中。现在我们仅仅打印每个作者：
    
    authors.runWith(Sink.foreach(println))

或者使用一个简短的版本（只被最热门的Sinks定义。如fold/foreach）
    
    authors.runForeach(println)
    
物化和运行流通常需要一个materializer在隐式作用域中（或者显式传递）
完全的代码片段如下：
    
    implicit val system = ActorSystem("reactive-tweets")
    implicit val materializer = ActorMaterializer()
     
    val authors: Source[Author, NotUsed] =
      tweets
        .filter(_.hashtags.contains(akkaTag))
        .map(_.author)
     
    authors.runWith(Sink.foreach(println))
    
##flattening sequences in streams
    
在之前的部分我们工作在1:1的元素关系上，通常大多数情况就是这样。但是有时候，我们也许希望将一个元素映射为一定数量的元素，然后收到
一个flatten的stream，像flatMap工作在scala集合上一样，为了获取一个包含hashTags的flattened stream,我们可以使用mapConcat这个
组合器。
    
    val hashtags: Source[Hashtag, NotUsed] = tweets.mapConcat(_.hashtags.toList)
    
注意：

    

