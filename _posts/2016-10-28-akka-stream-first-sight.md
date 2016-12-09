---
layout: default
title: 初识Akka Stream
---

## Introduction
本文主要翻译了*akka-stream*的官方文档中的相关基础介绍部分，并加上笔者使用过程中的部分体验。
## Motivation
我们现在从互联网上获取的服务的方式中包含很多流式数据处理的实例，如从一个服务中进行下载和上传，或者点到点（p2p）数据传输。把数据当作一个包含很多元素的流，而不是一个整体，
这种想法非常有用，因为这种想法符合电脑发送和收到数据的方式（例如通过tcp传输），但是它通常也是一种必要条件，因为数据集通常变得很大而不能作为一个整体被处理。我们通过
一个大集群将其分开计算或者解析，并称之为“大数据”，处理它们的总体准则是将这些数据按顺序的-作为一个流-通过一些cpu处理。   
*actors*看起来能够一样优秀的处理流式数据。它们按顺序发送和接收一系列的消息为了将知识（或数据）从一个地方传到到另外一个地方。我们发现它乏味和容易出错的在*actor*之间去获取稳
定的流通过按顺序实现所有正确的操作。这是因为为了发送和接收消息，我们也需要注意不要溢出内存或者进程中的*mailbox*。另外一个陷阱是*actor*消息可能会丢失，并一定会重传，为了避免在那种情况下，
流在获取数据端踩坑。当处理有很多被给定类型的元素数据流时。*actors*现在也不能提供一个很好的稳定的保证没有连线错误会被生成，类型安全能在这种情况下提升。  
由于这些原因我们决定对这些问题打包一个解决方案作为*Akka-Stream-Api*。这样做的目的是为了提供一个直观和安全的方式去制定流式操作处理，这样我们能接着能高效率的处理数据和使用相关资源，
再也不会有内存溢出的错误。为了达到这个目的，我们的流需要能够限制它们使用的内存，它们需要能够降低生产者的速率如果消费者不能跟上节奏。这个特性被称作
背压/反向压力(**back-pressure**)，这也是响应式流(*akka*是基础成员)的核心倡议。对于你，这意味着传播和对于*back-pressure*的响应难题已经解决在了*akka-stream*的设计中，
所以你可以少考虑一件事情。这也意味着*akka-stream*可以和其他响应式流的实现（响应式流的接口定义了互操作SPI,例如*akka-stream*这样的实现提供了一个优雅的使用者API）进行无缝互操作。

## Quick Start

现在我们从一个普通的例子开始，生成1到100的数字

    val source: Source[Int, NotUsed] = Source(1 to 100)

*source*类型被两个类型参数限定，第一个是*source*生成的元素的类型，第二个也许标识着运行*source*时生成了一些副作用值（例如，一个网络数据源也许会
提供绑定端口或者另一端的地址信息）。当没有副作用信息被生成时，类型*akka.NotUsed*会被使用，一个通常的数字范围肯定属于这个范畴。   

已经创造了这个数据源表示着我们有了一个如何生成前100个自然数的描述，但是这个源并不活跃。为了让这些数字出来，我们不得不运行它：

    source.runForeach(i => println(i))(materializer)
    
这一行代码会通过一个消费方法结束这个数据源，这个例子中我们仅仅打印这些数字到控制台，然后传递这个小*stream*到启动一个能运行它的*Actor*上，
这个动作通过使*run*成为方法名的一部分来标识，这里还有其他的方法运行*akka streams*,如下面这部分所示。  
你也许会好奇运行*stream*的*Actor*从哪里被创建,你也许也会问你自己这个*materializer*意味着什么。为了得到答案我们需要先创建一个*actor-system*。
    
    implicit val system = ActorSystem("QuickStart")
    implicit val materializer = ActorMaterializer()
    
这里还有其他的方法来创建*materializer*，例如，当在*actors*内部使用*streams*时从一个*ActorContext*中创建。*Materializer*是一个流处理引擎的工厂，
正是这个东西使流运行，你不必担心任何详细的细节除了你需要使用一个*materializer*为了在一个*source*上执行*run*方法。这个*materializer*被隐式的调用如果
*run*方法需要参数时，我们接着也会这样做。  
*akka-stream*的一个好处是*source*只是一个你想要运行的内容的描述，像一个建筑师的蓝图一样它能被复用，去合并到一个更大的设计中去。我们也许会选择转换
*source*中的数字然后写入到一个文件中：

    val factorials = source.scan(BigInt(1))((acc, next) => acc * next)
     
    val result: Future[IOResult] =
      factorials
        .map(num => ByteString(s"$num\n"))
        .runWith(FileIO.toPath(Paths.get("factorials.txt")))
        
首先我们使用*scan*这个合并者在整个流上运行一个计算:以数字1开始，我们乘以每个输入进来的数字，一个接着一个，这个*scan*操作提交初始值和每个
计算结果，这生成了一系列阶乘数字我们藏匿到*source*中为了以后的使用，这很重要记住--没有任何东西已经实际上被计算了。这只是一个当我们运行
这个流时我们想执行一次的计算的描述。然后我们转换这一系列的数字结果到一个流包含了*ByteString*对象来描述一个*text*文件中的行。这个流接着
通过连接一个文件运行，作为数据的接收者。在*akka-stream*的科技中这被叫做*sink*（水槽）。*IOResult*是*akka-stream*中*IO*操作的一个返回类型，
用来告诉你多少比特或者元素被消费，这个流的中断是正常还是异常。
       
##Reusable Pieces
       
*akka-stream*一个很好的部分就是（其他流式库并不提供的）不仅仅*source*能被重用作蓝图，所有其他的元素也能够被这样使用。我们能拿到文件，
写入*Sink*，前缀操作步骤必须去获取*ByteString*元素从输入的字符串和包中的操作也能成为被重用的部分。因为写这些流的语言通常从左边流到右边，
就像严肃的英语，我们需要一个起始点就像一个*source*，但是带着一个开放的输入。在*akka-stream*中，叫做*flow*:
      
      def lineSink(filename: String): Sink[String, Future[IOResult]] =
        Flow[String]
          .map(s => ByteString(s + "\n"))
          .toMat(FileIO.toPath(Paths.get(filename)))(Keep.right)
          
从一个字符串的*flow*开始，我们把每个元素转换成*ByteString*,然后将其传入已知的写文件的*sink*.这个结果的蓝图是一个*Sink[String,Future[IOResult]]*,
这意味着它接受一个字符串作为输入，当物化（*materialized*）时会生成类型为*Future[IOResult]*副作用信息（当在一个*source*上执行链式操作或者*flow*这个类型的副作用
称作物化值(*materialized value*)-被最左边的起始点给出，由于我们想保留*FileIO.toPath*的副作用，*sink*不得不提供，我们需要描述的*Keep.right*)。  
我们能使用我们刚刚创造的崭新的*Sink*,通过连接它到我们的阶乘源，通过一小段适配后去将数字转化为字符串:
          
       factorials.map(_.toString).runWith(lineSink("factorial2.txt"))
       
##Time-Based Processing
       
当我们看一个更有参与度的例子之前，我们探讨下*akka-stream*能做的流式处理的本质。从阶乘元素数据源开始，我们通过和另外一个流进行拉链操作来转换这个流，
通过一个数据源提供0-100的数字，第一个数字被阶乘源提交的是阶乘0，第二个是1的阶乘，我们合并这两个通过生成字符串如“3！=6”
    
           val done: Future[Done] =
             factorials
               .zipWith(Source(0 to 100))((num, idx) => s"$idx! = $num")
               .throttle(1, 1.second, 1, ThrottleMode.shaping)
               .runForeach(println)
 
到目前为止的所有的操作是时间依赖的，集合的元素能够被严格的呈现为相同的特性。接下来一行演示了我们实际上处理能够以一个理想的速度流动的数据流，
我们使用*throttle*合并者来降低流的速度到1s一个元素（这个参数列表中的1秒是我们想要允许迸裂(*burst*)的最大值，传递1意味着第一个元素马上通过后
接着的第二个不得不等待一秒，其他的以此类推）    
               
如果你运行这个程序，你会看到每秒会打印一行。一方面并不是立即可见需要引起注意，尽管如此，如果你试着设置流去生成数十亿记的数字，然后
你会注意到JVM不会因为内存溢出而崩溃，甚至你也会注意运行着的流，在后台运行，异步（这是为什么副作用信息以*future*的形式提供的原因）。使之能够
这样工作的秘密是*akka-stream*隐式的实现了普遍（*pervasive*）的流（*flow*）控制，所有的合并者（*combinators*）遵守反向压力/背压（*back-pressure*）规则。
这样就允许限流合并者去发信号给它的所有上游的持有数据的源，它只能在一个合理的速率下接收数据。当输入速率高于每个一秒，这个限流合并者会断言
反向压力给上游（**assert back-pressure upstream**）      

这些就是*akka-stream*在一个坚果壳中的基础，事实上的光泽会有几打的*Sources*和*Sinks*和许多更多的流转换合并者去选择.

## reactive tweets

对数据流进行操作的一个经典用法是消费一个我们想从中提取或合并数据的的流.在这个例子中，我们会考虑消费*tweets*的流，然后着重于用*akka*
从中提取信息。    
我们也会考虑到所有无阻塞流解决方案的固有问题：如果订阅者太慢而无法正常消费数据。通常这种解决方案是把这些元素放入内存，但是这样做
通常会最终引起的内存溢出和这类系统的不稳定性。*akka stream*相反，依赖于内部的反向压力（*back-pressure*）信号这样就允许控制在这样的场景下应该发生什么。    
下面是我们将要工作的数据模型，通过*quick start*这一节的例子看：

    final case class Author(handle: String)
     
    final case class Hashtag(name: String)
     
    final case class Tweet(author: Author, timestamp: Long, body: String) {
      def hashtags: Set[Hashtag] =
        body.split(" ").collect { case t if t.startsWith("#") => Hashtag(t) }.toSet
    }
     
    val akkaTag = Hashtag("#akka")
    
## transforming and consuming simple streams    
    
通过这个应用示例，我们看到的是一个普通的*Twitter*输入流，我们希望从中获取需要的信息，例如找到所有发*tweet*以*#akka*开头的用户。
为了准备我们的环境通过创建一个*actor-system*和*actor-materializer*，这些是我们将要创造的物化和运行*streams*所需要的：
    
    implicit val system = ActorSystem("reactive-tweets")
    implicit val materializer = ActorMaterializer()
    
这个*ActorMaterializer*能够接收一个*setting*作为参数（能被用来定义*materializer*的属性），例如默认的内存大小，这个分发器也被用来
管线化*pipeline*。这些配置能在*Flow*,*Source*,*Sink*和*Graph*上使用*withAttributes*方法来重写。   
假设我们有一个*tweets*流已经可用了，在*akka*中被表示为*Source[Out,M]*

    val tweets: Source[Tweet, NotUsed]
    
流通常开始流动从*Source*开始，然后接着流过*Flow*,或者更多更高级的*graph*元素，最后被*sink*消费掉。
这些操作看起来和所使用的任何一个*scala*集合库一样，然而它们操作于流之上，而不是一个数据集。（这是非常重要的区别，因为一些操作只在流上有意义，反之亦然）
    
    val authors: Source[Author, NotUsed] =
      tweets
        .filter(_.hashtags.contains(akkaTag))
        .map(_.author)

最后为了物化和运行流计算，我们需要绑定*flow*到*Sink*上是的*flow*得以运行。最简单的做法是在一个*source*上使用*callWith(sink)*方法。为了简便，
一系列通常的*sinks*被预先定义了，然后作为方法收集起来放在了*sink*的伴生对象中。现在我们仅仅打印每个作者：
    
    authors.runWith(Sink.foreach(println))

或者使用一个简短的版本（只被最热门的*Sinks*定义。如*fold*/*foreach*）
    
    authors.runForeach(println)
    
物化和运行流通常需要一个*materializer*在隐式作用域中（或者显式传递）
完全的代码片段如下：
    
    implicit val system = ActorSystem("reactive-tweets")
    implicit val materializer = ActorMaterializer()
     
    val authors: Source[Author, NotUsed] =
      tweets
        .filter(_.hashtags.contains(akkaTag))
        .map(_.author)
     
    authors.runWith(Sink.foreach(println))
    
##flattening sequences in streams
    
在之前的章节我们工作在1:1的元素对应关系上，通常大多数通常情况就是这样。但是有时候，我们也许想将一个元素映射为一定数量的元素，然后收到
一个达到打倒(*flattened*)的*stream*，就像*flatMap*工作在*scala*集合上一样，为了获取一个包含*hashTags*的*flattened stream*,我们可以使用*mapConcat*这个
组合器。
    
    val hashtags: Source[Hashtag, NotUsed] = tweets.mapConcat(_.hashtags.toList)
    
##broadcasting a stream

我们现在想持久化所有的*hash-tags*，同样也持久化所有作者名字从这个活的*stream*。例如我们乐意去将所有的作者的句柄写进一个文件，所有*hash-tags*
写入硬盘中的另外一个文件。这意味着我们不得不将一个数据源流分割为两个流来处理写入两个不同的文件。      

被用于生成这样的扇出（*fan-out*）或者扇入(*fan-in*)结构的元素在*akka-stream*中被称作胶水/连接剂(*junctions*).其中我们将要使用在这个例子中的叫做广播
*Broadcast*。它只是简单的提交元素从输入端口到所有输出端口。    

*akka-streams*内部分开了线性的流结构（*Flow*）从非线性的分支（*Graphs*）为了提供最方便的API在着两种情况下。*Graphs*能描述任意复杂的流设置，
在以不读取熟悉的集合转换的开销下(*at the expense of not reading as familiarly as collection transformations*)。

*Graphs*被构造使用*GraphDSL*:

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

如你所见，在*GraphDSL*中，我们使用隐式的*graph*建造者b使用**~>**"边缘操作符"（也叫做connect/via/to去改变构造*graph*.这个操作符通过引入`GraphDSL.Implicits._`来被
隐式提供。    
   
*GraphDSL.create*返回一个*graph*，例如*Graph[ClosedShape，Unit]*,*ClosedShape*意味着这个*graph*被完全连接或者关闭，这里没有未连入的输入或输出。因为
它是关闭的，所以通过使用*RunnableGraph.fromGraph*是可以将*graph*转换到*RunnableGraph*。这个可以运行的*graph*然后能被*run()*方法执行去物化一个流。   
 
*Graph*和*RunnableGraph*都是不可变的，线程安全的，可被任意分享的。
    
一个*graph*能拥有一个其他的类型，一个或者多个未连接的*ports*.有未连接的*ports*表述一个*graph*是一个"偏-图"(*partial-graph*)。这个概念围绕组成和嵌套
graphs变成一个巨大的数据结构，被详细介绍在稍后的文章中。这也使包裹复杂计算的*graphs*作为*Flows*,*Sinks*,*Sources*成为可能，也会在后面的文章介绍。
    
    
## back-pressure in action
   
*akka-stream*的一个主要优势他们允许传播*back-pressure*信息从流的*Sinks*(订阅者)到*Sources*(发布者)。这不是一个可有可无的特性，这被允许在所有的时刻。
为了学习更多*akka-stream*使用的*back-pressure*的实现日后会提及。
   
一个(不使用*akka-stream*)应用遇到的经典的问题，他们不能足够快的处理输入的数据，不管是临时的或者设计上的，这会导致开始将输入的数据放入内存中，直到没有
内存空间，结果是造成内存溢出错误或者其他服务响应降解（*degradations*）。通过*akka-streams*使用缓冲能而且必须显示的处理。例如，如果我们仅仅对最新的*tweeter*感兴趣，
缓冲10个元素。这能被使用*buffer*元素表示：

    tweets
      .buffer(10, OverflowStrategy.dropHead)
      .map(slowComputation)
      .runWith(Sink.ignore)
      
这个*buffer*元素带着一个显式的和必须的*OverflowStrategy*，这定义了*buffer*当它满的时候接收到了一个元素该如何处理新进入的元素。策略提供了包括扔掉最老的元素（*dropHead*）,
扔掉多余的*buffer*，显示错误等方式。确定和选择最适合你的使用场景的策略。
      
##Material values
      
到现在为止，我们仅仅使用*Flows*处理了数据，然后消费数据到外部的*sink*-打印出来或者存储到外部的系统。然而有时候我们也许对一些能从*materialized*中获取，能被
管线化（*pipeline*）处理的数据更感兴趣。例如，我们想要知道我们处理了多少*tweets*.这个问题的答案并不好明显的给出答案，因为无限的*tweets*流（一个解决这个问
题的方法是在设置的流中创造一个流描述到目前为止，我们处理了多少条*tweets*）。但是通常的处理有限的流是可能的，然后获得一个很好的答案，如元素总数。   
首先，我们使用*Sink*,*fold*写一个计数的元素，来看类型看起来会怎样:
      
      val count: Flow[Tweet, Int, NotUsed] = Flow[Tweet].map(_ => 1)
       
      val sumSink: Sink[Int, Future[Int]] = Sink.fold[Int, Int](0)(_ + _)
       
      val counterGraph: RunnableGraph[Future[Int]] =
        tweets
          .via(count)
          .toMat(sumSink)(Keep.right)
       
      val sum: Future[Int] = counterGraph.run()
       
      sum.foreach(c => println(s"Total tweets processed: $c"))
      
首先我们准备了一个可复用的*Flow*，它会改变每个进入的*tweet*变成1，我们会使用这个，为了合并那些使用*sink*，*fold*的操作，这会求和流中的所有的int元素。然后使
计算结果变得可用作为一个*Future[Int]*.接着我们连接*tweets*流到*count*通过一个*via*,最后我们使用*toMat*连接*Flow*到原先的准备好的*Sink*。    
     
记住这些神秘的*Mat*类型参数使用在 *Source[+Out, +Mat], Flow[-In, +Out, +Mat],Sink[-In, +Mat]* 。它们代表着这些处理部分返回的值的类型当被物化时，
当你把它们串起来时，你能显式的合并它们物化的值。在我们的例子中，我们使用了*Keep,right*这些预定义的方法，这表示实现只需要关心当前情境链接到后面的
物化类型。这个*sumSink*的物化类型是 *Future[Int]*,因为使用了*keep.right*。*RunnableGraph*的结果也会是一个 *FUture[Int]*的类型。    
  
这一步还没有物化处理的*pipeline*，它只是准备了一个链接到*sink*的*Flow*的描述，因此能被执行*run*方法，也被它的类型暗示：*RunnableGraph[Future[Int]]]* .现在
我们使用*run*方法，会使用隐式的使用*ActorMaterializer*来实行物化和运行*Flow*，这个通过*run*方法在*RunnableGraph[T]*上运行的返回值类型为T。在我们的例子中是
*Future*.当结束时，会包含我们的*tweets*流的总长度。在流运行失败的场景下，*future*会以一个*Failure*结束。    

一个*RunnableGraph*能被复用和物化多次，因为它仅仅是流的蓝图。这意味着如果我们物化一个流，例如消费一个活的流在一分钟之内，这个物化的值对于那两个物化会不同。
如下面例子:
 
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
    
许多akka-streams中的元素提供物化值，能被用于包含其它的计算结果或者操舵(*steering*)这些元素在下个文章中详解。合并这一小块，现在我们知道当我们运行这个线性处理的例子背后
发生了什么，这和多个线性操作的版本一致。
    
    val sum: Future[Int] = tweets.map(t => 1).runWith(sumSink)
    
Note:
*runWith()*是一个非常方便的方法会自动忽略其他阶段的物化值除了那些连接着*runWith()*操作本身的操作。在上述的例子中它转换成使用*keep.right*作为合并物化值。 

## Conclusion
*akka-stream*是基于*akka*开发的用于流式数据处理的响应式流(*reactive-stream*),响应式的核心在于内置的传递了消费者的*back-pressure*,以避免
消费者消费过慢而导致消费者端的应用崩溃。    
总体上看可以看作水源/数据源(*source*)经过流动(*flow*)最终流到水槽(*sink*),我们可以在流动的过程中对数据进行相应的操作和转换，最终进行消费。    
所有的操作在*run*之前，都只是对数据操作方式的一种描述。需要被物化(*materialzed*)和运行(*run*)后，数据才会真正的流动起来。   
自身定制了一些*dsl*可以方便的进行流的合并,拆分，转换以及各种操作，同时*graph*可以使我们组合成相当复杂的流，亦可以轻松的和其他的响应式流进行响应的交互。

               
               
          