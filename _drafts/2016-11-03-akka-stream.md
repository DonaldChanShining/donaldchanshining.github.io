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
    
##broadcasting a stream

我们现在想持久化所有的hash-tags，同样也持久化所有作者名字从这个live的stream。例如我们乐意去将所有的作者的句柄写进一个文件，所有hashtags
写入硬盘中的另外一个文件。这意味着我们不得不将原来的一个流分割为两个流来处理写入两个不同的文件。

被用于生成这样的扇出（fan-out）或者扇入结构的元素在akka-stream中被称作胶水/连接剂（junctions）.其中我们将要使用在这个例子中的叫做广播
Broadcast。它只是简单的提交元素从输入端口到所有输出端口。

akka-streams内部分开了线性的流结构（Flow）从非线性的分支（Graphs）为了提供最方便的API在着两种情况下。Graphs能描述任意复杂的流设置，
在不和集合转换类似读取的开销下。

Graphs被构造使用GraphDSL.

    val writeAuthors: Sink[Author, Unit] = ???
    val writeHashtags: Sink[Hashtag, Unit] = ???
    val g = RunnableGraph.fromGraph(GraphDSL.create() { implicit b =>
      import GraphDSL.Implicits._
     
      val bcast = b.add(Broadcast[Tweet](2))
      tweets ~> bcast.in
      bcast.out(0) ~> Flow[Tweet].map(_.author) ~> writeAuthors
      bcast.out(1) ~> Flow[Tweet].mapConcat(_.hashtags.toList) ~> writeHashtags
      ClosedShape
    })
    g.run()

如你所见，在GraphDSL中，我们使用隐式的graph b去变换构造graph使用在边缘操作者（也叫做 connect/via/to）.这个操作者通过引入import. ....来被
隐式提供。
   
GraphDSL创建后返回一个graph，例如*Graph[ClosedShape，Unit]*,ClosedShape意味着这个graph被完全连接或者关闭，这里没有未连入的输入或输出。因为
它是关闭的，所以可能转换graph到RunnableGraph使用RunnableGraph通过fromGraph。这个可以运行的graph然后被run去物化一个流。
    
Graph和RunnableGraph都是不可变的，线程安全的，可被任意分享的。
    
一个graph能拥有一个其他的类型，一个或者更多的未连接的Ports.有未连接的ports表述一个graph是一个偏-graph(partial-graph)。这个概念围绕组成和嵌套
graphs变成一个巨大的数据结构，被详细介绍在稍后的文章中。这也使包裹复杂计算的graphs作为FLOW,SINK,SOURCE成为可能，也会在后面的文章介绍。
    
    
##bakckpressure in action
   
akka-stream的一个主要优势他们允许传播back-pressure信息从流的Sinks(订阅者)到Sources(发布者)。这不是一个可有可无的特性，这被允许在所有的时刻。
为了学习更多akka-stream使用的back-pressure协议读后面的文章。
   
一个应用（不使用Akka-stream）遇到的经典的问题，他们不能处理输入的数据足够快，不管是临时的或者设计上的，这会开始将输入的数据放入内存中，直到没有
内存空间，结果是造成内存溢出错误或者其他服务响应降解。使用akka-streams使用内存能而且必须被显示处理。例如，如果我们仅仅对最新的tweeter感兴趣，
Buffer10个元素。这能被如下表述：

    tweets
      .buffer(10, OverflowStrategy.dropHead)
      .map(slowComputation)
      .runWith(Sink.ignore)
      
这个Buffer元素带着一个显式的和必须的OverflowStrategy，这定义了Buffer该如何响应当它满的时候接收到了一个元素。策略提供了包括扔掉最老的元素（dropHead）,
扔掉多余的Buffer，显示错误等方式。确定选择最适合你的使用场景的策略。
      
##Material values
      
到现在为止，我们仅仅使用FLOW处理了数据，然后消费数据到sink的延伸，打印出来或者存储到其他的系统。然而有时候我们也许对一些能从materialized中获取，能被
管道化（pipeline）处理的数据更感兴趣。例如，我们想要知道我们处理了多少tweets.这个问题的答案并不好明显的给出答案，因为无限的tweets流（一个解决这个问
题的方法在一个设置的流中是创造一个流描述到目前为止，我们处理了多少条tweets）。但是通常的处理有限的流是可能的，然后获得一个很好的答案，如元素总数。

首先，写如下代码：
      
      val count: Flow[Tweet, Int, NotUsed] = Flow[Tweet].map(_ => 1)
       
      val sumSink: Sink[Int, Future[Int]] = Sink.fold[Int, Int](0)(_ + _)
       
      val counterGraph: RunnableGraph[Future[Int]] =
        tweets
          .via(count)
          .toMat(sumSink)(Keep.right)
       
      val sum: Future[Int] = counterGraph.run()
       
      sum.foreach(c => println(s"Total tweets processed: $c"))
      
首先我们准备了一个可复用的Flow，它会改变每个进入的tweet变成1，我们会使用这个，为了合并那些使用sink，fold的操作，这会求和流中的所有的int元素。然后使
计算结果变得可用作为一个Future[Int].接着我们连接tweets流到count通过一个via,最后我们使用toMat连接Flow到原先的准备好的Sink。
     
记住这些神秘的Mat类型参数使用在 *Source[+Out, +Mat], Flow[-In, +Out, +Mat],Sink[-In, +Mat]* 。它们代表着这些处理部分返回的值的类型当被物化时，
当你把它们串起来时，你能显式的合并它们物化的值。在我们的例子中，我们使用了Keep,right这些预定义的方法，这表示实现只需要关心当前情境链接到后面的
物化类型。这个sumSink的物化类型是 *Future[Int]*,因为使用了keep.right。RunnableGraph的结果也会是一个 *FUture[Int]*的类型。
  
这一步还没有物化处理的pipelien，它只是准备了一个链接到sink的Flow的描述，因此能被执行run方法，也被它的类型暗示：*RunnableGraph[Future[Int]]]* .现在
我们使用run方法，会使用隐式的使用ActorMaterializer来实行物化和运行Flow，这个通过run方法在RunnableGraph[T]上运行的返回值类型为T。在我们的例子中是
Future.当结束时，会包含我们的tweets流的总长度。在流运行失败的场景下，futrue会以一个Failue结束。

一个RunnableGraph能被复用和物化多次，因为它仅仅是流的蓝图。这意味着如果我们物化一个流，例如消费一个活的流在一分钟之内，这个物化的值对于那两个物化会不同。
如下面例子中的分别。
 
    val sumSink = Sink.fold[Int, Int](0)(_ + _)
    val counterRunnableGraph: RunnableGraph[Future[Int]] =
      tweetsInMinuteFromNow
        .filter(_.hashtags contains akkaTag)
        .map(t => 1)
        .toMat(sumSink)(Keep.right)
     
    // materialize the stream once in the morning
    val morningTweetsCount: Future[Int] = counterRunnableGraph.run()
    // and once in the evening, reusing the flow
    val eveningTweetsCount: Future[Int] = counterRunnableGraph.run()
    
许多akka-streams中的元素提供物化值，能被用于包含其它的计算结果或者操舵这些元素在下个文章中详解。合并这一小块，现在我们知道当我们运行这个线性处理的例子背后
发生了什么，这和多个线性操作的版本一致。
    
    val sum: Future[Int] = tweets.map(t => 1).runWith(sumSink)
    
Note:
runWith()是一个非常方便的方法会自动忽略其他阶段的物化值除了那些连接着runWith()操作本身的操作。在上述的例子中它转换成使用keep.right作为合并物化值。      
      

