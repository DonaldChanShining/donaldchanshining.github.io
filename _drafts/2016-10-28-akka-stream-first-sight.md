---
layout: default
title: Akka Stream Learning Summary
---

一个流(stream)通常从一个源（source）开始聊，这也是我们开始学习akka-stream的方式。在我们创造一个之前，我们引入streaming的工具所有的实现。

    import akka.stream._
    import akka.stream.scaladsl._
    
如果你想执行样例代码当你阅读这个快速开始指导时，你也需要下列引用

    import akka.{ NotUsed, Done }
    import akka.actor.ActorSystem
    import akka.util.ByteString
    import scala.concurrent._
    import scala.concurrent.duration._
    import java.nio.file.Paths

现在我们从一个很普通的例子开始，生成1到100的数字

    val source: Source[Int, NotUsed] = Source(1 to 100)

*Soucrce*类型被两个类型参数限定，第一个是source生成的元素的类型，第二个也许标识着运行source时生成了一些副作用值（例如，一个网络source也许会
提供绑定端口或者另一端的地址信息）。当没有副作用信息被生成时，类型*akka.NotUsed*会被使用，一个普通的数字范围肯定属于这个范畴。
已经创造了源表示着我们有了一个如何生成100个自然数的概念，但是这个源并不活跃。为了让这些数字出来，我们不得不运行它：

    source.runForeach(i => println(i))(materializer)
    
这一行会通过一个消费者方法完成（complement）这个源，这个例子中我们仅仅打印这些数字到控制台，然后传递这一点点Stream到启动一个能运行它的Actor上，
这个动作通过使*run*成为方法名的一部分来标识，这里还有其他的方法运行Akka Streams,如下面这部分所示。
你也许会好奇Actor从哪里被创建，运行Streams,你也许也会问你自己这个materializer意味着什么。为了解决这个问题我们先创建一个actor-system。
    
    implicit val system = ActorSystem("QuickStart")
    implicit val materializer = ActorMaterializer()
    
这里还有其他的方法来创建要给materializer，例如，从一个ActorContext中当在actors中使用Streams时。这个materializer是一个流处理引擎的工厂，
正是这个东西使流运行，你不必担心任何详细的细节除了你需要使用一个materializer来在一个source上执行run方法。这个materializer被隐式的调用如果
它被给run方法需要参数，我们会接着这样做。
AkkaStream的一个好处是Source只是一个你想要run的内容的描述，像一个建筑师的蓝图一样它能被复用，去合并到一个更大的设计中去。我们也许会选择转换
source中的数字然后写入到一个文件中：

    val factorials = source.scan(BigInt(1))((acc, next) => acc * next)
     
    val result: Future[IOResult] =
      factorials
        .map(num => ByteString(s"$num\n"))
        .runWith(FileIO.toPath(Paths.get("factorials.txt")))
        
首先我们使用scan合并者去运行一个计算在整个流上，以数字1开始，我们乘以每个输入进来的数字，一个接着一个，这个scan操作提交初始值和每个
计算结果，这生成了一系列阶乘数字我们藏匿到Source中为了以后的使用，这很重要记住--没有任何东西已经实际上被计算了。这只是一个当我们运行
这个流时我们想执行一次的计算的描述。然后我们转换这一系列的数字结果到一个流包含了ByteString对象来描述一个text文件中的行。这个流接着
通过连接一个文件运行，作为数据的接收者。在akka-stream的科技中这被叫做sink（水槽）。IOResult是AKKA-stream中IO操作的一个返回类型，
用来告诉你多少比特或者元素被消费，这个流的中断是正常还是异常。
       
##可重用部分
       
Akka-Stream一个很好的部分就是（其他stream库并不提供的）不仅仅Sources能被重用作蓝图，所有其他的元素也能够被这样使用。我们能拿到file，
写Sink，前缀操作步骤必须去获取ByteString元素从输入的Strings和包中的操作也能成为被重用的部分。因为写这些流的语言通常从左边流到右边，
就像严肃的英语，我们需要一个起始点就像一个source，但是带着一个开放的输入。在Akka-Stream中，叫做flow:
      
      def lineSink(filename: String): Sink[String, Future[IOResult]] =
        Flow[String]
          .map(s => ByteString(s + "\n"))
          .toMat(FileIO.toPath(Paths.get(filename)))(Keep.right)
          
从一个String的flow开始，我们转换每个元素到ByteString,然后将其喂入已知的写文件的Sink.这个结果的蓝图是一个Sink[String,Future[IOResult]],
这意味着它接受一个String作为输入，当物化它时会生成类型为Future[IOResult]副作用信息（当在一个source上执行链式操作或者Flow这个类型的副作用
称作“materialized value”-被最左边的起始点给出，由于我们想保留FileIO.toPath提供的，我们需要说Keep.right）
我们能使用一个新的闪亮的Sink，我们刚刚创造的，通过连接它到我们的阶乘源，通过一小段适配后去将数字转化为Strings
          
       factorials.map(_.toString).runWith(lineSink("factorial2.txt"))
       
##时序处理
       
当我们看一个更有参与度的例子之前，我们看看Akka-Stream能做的流式处理的本质。从阶乘元素源开始，我们通过和另外一个流进行拉链操作来转换一个流，
被一个源提供数字0-100，第一个数字被阶乘源发射的是阶乘0，第二个是1的阶乘，我们合并这两个通过生成字符串如“3！=6”
    
           val done: Future[Done] =
             factorials
               .zipWith(Source(0 to 100))((num, idx) => s"$idx! = $num")
               .throttle(1, 1.second, 1, ThrottleMode.shaping)
               .runForeach(println)
 
到目前为止所有的操作成为需要时间，集合的元素能够被严格的呈现为相同的特性。接下来一行演示了我们实际上处理流，能够以一个速度流动，
我们使用throttle合并者来降低流的速度到1S一个元素（这个参数列表中的1秒是我们想要允许爆炸的最大值，传递1意味着第一个元素马上通过后
接着的第二个不得不等待一秒，其他的以此类推）
               
如果你运行这个程序，你会看到每秒打印一行。一方面着并不是立即可见需要引起注意，尽管如此，如果你试着设置流去生成数十亿记的数字，然后
你会注意到JVM不会因为内存溢出而崩溃，甚至你也会注意运行着的流，在后台运行，异步（这是因为副作用信息以future的形式提供）。使之能够
这样工作的秘密是AKKA-STREAM隐式的实现了普遍（pervasive）的流（flow）控制，所有的合并者（combinators）遵守背压（back-pressure）规则。
这样就允许限流合并者去发信号给它的所有上游的持有数据的源，它只能接收数据在一个合理的速率。当输入速率高于每个一秒，这个限流合并者会断言
背压上游（assert back-pressure upstream）

这些就是akka-stream在一个坚果壳中的基础，事实上的光泽会有几打的Sources源和Sinks和许多更多的流转换合并者去选择。看文档abcd>.

##响应式消息


               
               
          