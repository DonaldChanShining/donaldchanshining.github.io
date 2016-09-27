---
layout: post
title: spray学习小结
---
 
此文为io.kamon上一篇对于[spray-client](http://kamon.io/teamblog/2014/11/02/understanding-spray-client-timeout-settings) 
的工作机制说明，翻译下来以供后来者方便理解其工作机制，对以后使用Akka-Http也大有裨益。

## 客户端配置说明

    spray.can {
      client {
        # 客户端连接在触发request-timeout前等待服务端响应的最大时长。这里的计时器不会开始计时
        # 直到连接处于正在接收响应的状态，也就是说计时可能始于后端应用接收请求后一段时间。
        # 延迟开始计时器的原因主要有如下两个：
        # 1. 在spray的host-level等级的API中不支持pipeline:
        #    如果请求因为所有连接都正在处理以前的请求不能够立即发送，那么这个请求不得不放在队列中，
        #    直到有一个连接变得可用。  
        # 2. 如果支持pipeline
        #    请求超时计时器在对正在处理的请求的响应于当前连接上返回时开始计时。   
        #    The request timeout timer starts only once the response for the
        #    preceding request on the connection has arrived.
        request-timeout = 20 s
    
        # tcp连接处理必须完成的时间
        connecting-timeout = 10s  
      }
    
      host-connector {
        # HttpHostConnector的最大并行连接数，建立一个长连接,必须大于0
        max-connections = 4
    
        # HttpHostConnector对于请求在可重试情况下的最大重试次数
        max-retries = 5
        
        # 如果开启此项设置，则httpHostConnector使用在连接通道上使用pipeline，不然
        # 在一个特定的Http连接上只能打开一个request
        pipelining = off  
      }
    }

## 使用Request-Level API

以一段普通的代码开始分析

    import akka.actor.ActorSystem
    import akka.util.Timeout
    import spray.client.pipelining.sendReceive
    import spray.httpx.RequestBuilding
    import scala.concurrent.duration._
    
    object SprayClientExample extends App with RequestBuilding {
      implicit val requestTimeout = Timeout(60 seconds)
      implicit val system = ActorSystem("spray-client-example")
      implicit val executionContext = system.dispatcher
    
      val clientPipeline = sendReceive
    
      val startTimestamp = System.currentTimeMillis()
      val response = clientPipeline {
        Get("http://127.0.0.1:9090/users/kamonteam/timeline")
      }
      response.onComplete(_ => println(s"Request completed in ${System.currentTimeMillis() - startTimestamp} millis."))
    }

假定我们已经拥有了一个服务监听了`localhost:9090`，会延时15s后响应进来的请求，启动上述的示例代码，短暂的等待后，会得到下面的输出：        

    Request completed in 16020 millis.

结果没什么特别，它只是按照我们所设想的去执行了。要注意我们对于 *Request-Level*的Api设置了一个较高的超时时间（60s）,主要是为了避免在
这个测试中阻塞我们，同时我们等下会重新访问。下面让我们开始令人惊讶的测试，如果我们使用默认配置的情况下一次性发送6个请求会发生什么？
下面是修改后的代码：

    for(r <- 1 to 6) {
        val startTimestamp = System.currentTimeMillis()
        val response = clientPipeline {
          Get("http://127.0.0.1:9090/users/kamonteam/timeline")
        }
        response.onComplete(_ => println(s"Request [$r] completed in ${System.currentTimeMillis() - startTimestamp} millis."))
      }

生成了如下输出(输出顺序可能不同当你自己测试的时候，但是最重要的部分还是保持相同)：
      
      Request [3] completed in 15887 millis.
      Request [4] completed in 15887 millis.
      Request [1] completed in 16033 millis.
      Request [2] completed in 15887 millis.
      Request [6] completed in 30891 millis.
      Request [5] completed in 30891 millis.

现在你可能在脑海中有两个疑问：首先，为何最开始的4个请求在15s左右完成，但是最后2个请求需要30s去完成？这个问题解释很简单，Spray Client
对于每一个你想发送请求的host创建一个 *HttpHostConnector*(粗略的描述，在实现上有更多细节)，你可以把一个host-connector当作一个连接池，拥
有`spray.can.host-connector.max-connections`配置的连接数，给予目标host默认连接数4，所以最开始的4个请求立即发送出去，其余2个请求需要
等待可用的连接。如果你将`max-connections`设置为一个更高的值，这样就会有足够的连接数来立刻分发所有的请求，所有的请求都会按我们的预期完成
在15s左右。  

第二个问题是，为何一个请求能够耗时30s去完成而`spray.can.client.request-timeout`是默认被设置为20s的？这个文档在开始说的很清楚，
**这里的计时器不会开始计时直到连接处于正在接收响应的状态，也就是说计时可能始于后端应用接收请求后一段时间**，但是这里的
**一段时间**是多长？好吧，这依赖于Spray Client工作引擎下的一些条件，通过这一小段很难弄清楚究竟发生了什么。    
![Alt text](http://donaldhome.com/images/spray.jpg)

下面描述一下上图中的流程：  
   
1.你的应用代码使用一个`sendReceive`来发送一个Http请求。默认情况下，`sendReceive`会使用 *HttpManager*作为一个Http请求的中继站(即当你
在actor-system中获取的`IO(Http)`的对象)。 *HttpManager*是spray框架相关的actor的根路径，同时在服务端和客户端，但是为了简介，我们只在
这里说明客户端相关的部分。   
2. *HttpManager*会使用在http请求中的目标host(其中包含其他细节)去决定http请求会发送到哪个 *HttpHostConnector*。如果对于目标host没有可用的
*HttpHostConnector*，则会创建一个，同时请求会被分发到这个 *HttpHostConnector*.此外，基于客户端连接设置(`user-agent header`, 
`idle timeout`, `request timeout`等等), *HttpManager*会找到或者创建一个被 *HttpHostConnector*所需要的 *HttpClientSettingsGroup*
来进行工作。    
3. *HttpHostConnector*使用了它的魔力。就像我们前面提到的， *HttpHostConnector*工作相当于一个连接池，它不得不找到或者创建一个合适的
*HttpHostConnectionSlot*来分发进来的http请求，如果不能找到， *HttpHostConnector*会将请求排队直到有一个合适的 *HttpHostConnectionSlot*
变成可用状态。根据HTTP-pipelining功能是否开启，请求分发的逻辑会不同。我们这里假设为默认设置`spray.can.host-connector.pipelining = off`,
你可以通过[spray官方文档](http://spray.io/documentation/1.2.3/spray-can/http-client/host-level/)来了解更多pipelining是如何工作的。
当host-connector创建一个新的 *HttpHostConnectionSlot*,新建的 *HttpHostConnectionSlot*具有 *HttpManager* 创建 *HttpHostConnector*
时传递的 *HttpClientSettingsGroup*。这就是我们先前的例子中最后两个请求入队等待连接可用的地方。  
4. *HttpHostConnectionSlot*会请求 *HttpClientSettingsGroup*来创建一个 *HttpClientConnection*,一当建立连接时 *HttpHostConnectionSlot*
就会分发请求给 *HttpClientConnection*。建立 *HttpClientConnection*的过程只会发生在第一个请求到达时，接下来相同的连接可以被复用
越长越好。  
5. *HttpHostConnection* 和 *Akka-IO*的底层进行交互来建立TCP连接并发送给其Http请求数据。这是异常简化的版本来描述这一步发生了什么，但是
为了保持简短，我们只需要了解它需要一定的时间来建立TCP连接，同时这段时间包含在`spray.can.client.connecting-timeout`指定的超时时间中，接
下来连接建立后，请求开始分发，`spray.can.client.request-timeout`开始计时。   

回头看问题 **一段时间**有多长，很多东西都有了概念和印象。如果没有可用的连接，请求会排队直到一个连接变成可用状态；如果没有连接建立，所有相
关联的actors会被创建。最后，从客户端应用代码发送请求到`request-timeout`开始计时的流程中至少有4条 *actor messages*参与，所有的这些actors，
都默认被 **Actor System**的调度器进行相关调度，但是调度器可能会忙于其他事务，而且有可能被一些幼稚的未分割的开销很大的操作阻塞。

## 关于Host Level API        

*Host-level API*直接和 *HttpHostConnector*进行会话而不是通过 *HttpManager*来完成。在这之后，所有步骤都和我们上面描述的一致，这里的要点
是你需要发送一条`Http.HostConnectorSetup`的消息给 *HttpManager*,然后它会回复 `Http.HostConnectorInfo`包含了 *HttpHostConnector*的
引用( *ActorRef*),这样你可以通过引用直接发送请求给 *HttpHostConnector*。   

如果你决定建立和直接使用 *HttpHostConnector*，一定记住它有一个闲置超时时间(`idle-timeout`)，超过了这个时间它会被停止，它的引用也将失效，
你可以选择将`idle-timeout`设置为无限或者 *watch*这个引用并优雅的处理这些场景。我们倾向于建议人们避免使用 *Host-level API*，因为使用它
会增加一些复杂度(那些已经被 *HttpManager*所解决的问题)，此外，在绝大多数的应用场景中，自行保留处理一条 *actor message*和所有需要处理的任务
耗时相比不会带来显著的性能提升。当然，每个示例有它自己的需求，只有通过测试你自己的app你才会知道。

## sendReceive Timeout的意义

看一下`sendReceive`的定义：

    type SendReceive = HttpRequest ⇒ Future[HttpResponse]
    
      def sendReceive(implicit refFactory: ActorRefFactory, executionContext: ExecutionContext,
        futureTimeout: Timeout = 60.seconds): SendReceive
    
      def sendReceive(transport: ActorRef)(implicit ec: ExecutionContext, futureTimeout: Timeout): SendReceive
      
      
在我们的例子中提供的隐式的超时是`requestTimeout`满足`sendReceive`方法的隐式参数`futureTimeout`，但是这个超时时间和所有的我们前面
提到的超时时间没有关系，它只是为了满足 *actor*的`ask`方法所需要的`timeout`参数，返回一个`Future[HttpResponse]`， *future*会被
填充当响应到达时。如果future的超时时间到了而没有收到响应，则Future会被填充为AskTimeoutException.但是请求会继续执行，直到有一个
结果返回。

## "继续执行"的意思

*Spray Client*并没有执行请求的截止日期的概念，它只知道请求得经过一系列的步骤去完成(不管成功或者失败)，只有经过所有步骤完成后，它才会
停止执行请求并返回一个响应。如果我们修改前面例子中的`requestTimeout`变为10s，所有响应的 *future*会在10s后完成为
*AskTimeoutException*，但是， *Spray Client*依然会将所有的请求发送到我们的服务，哪怕最后2个请求需要等待15s后连接变为可用状态。修改
过后生成的输出会和下面类似：
    
    Request [1] completed in 10161 milliseconds.
    Request [2] completed in 10020 milliseconds.
    Request [3] completed in 10020 milliseconds.
    Request [4] completed in 10020 milliseconds.
    Request [5] completed in 10020 milliseconds.
    Request [6] completed in 10020 milliseconds.
    
但是大约5s过后，当前4个response到达后会进入 *dead letters*因为没 *actor*在等待它们，大约再等15s后，最后2个响应到达，一样也会进入
*dead letters*,输出日志会和下面类似：
    
    [06:00:12.153] [akka://spray-client-example/deadLetters] Message [spray.http.HttpResponse] from ...
    [06:00:12.153] [akka://spray-client-example/deadLetters] Message [spray.http.HttpResponse] from ...
    [06:00:12.154] [akka://spray-client-example/deadLetters] Message [spray.http.HttpResponse] from ...
    [06:00:12.154] [akka://spray-client-example/deadLetters] Message [spray.http.HttpResponse] from ...
    [06:00:27.161] [akka://spray-client-example/deadLetters] Message [spray.http.HttpResponse] from ...
    [06:00:27.161] [akka://spray-client-example/deadLetters] Message [spray.http.HttpResponse] from ...
    
## 重试机制

最后一件你需要记住的事情是 *Spray Client*会默认重试幂等请求`spray.can.host-connector.max-retries`指定的次数，同时重试也是前面
提到的请求执行步骤中的一部分，也就是说 *Spray Client*在放弃之前会默认重试5次，不管你是否喜欢。如果你有一个服务端在另一边返回响应，
则它可能至少耗时100s(5次重试，20s每次，不包括连接建立时间)来使 *Spray Client*放弃一个请求。不设置重试次数取决于你的应用的需要，可
能会导致你的系统耗时执行那些你已不需要的请求。你可以认为一个真正的 *spray client*超时时间包括：
  
*   `spray.can.client.request-timeout` 对于幂等请求，乘以重试次数 `spray.can.host-connector.max-retries`，
    或者对于非幂等请求乘以1
*   `spray.can.client.request-timeout` 乘以 `spray.can.host-connector.max-redirects`。重定向次数默认是0，但是如果你将其
    配置为一个更大的值，请记得重定向次数计数独立于重试次数
*   可能花费在 *HttpHostConnector*的队列中等待可用的连接的时间，这个时间不能被任何配置项限制，而且唯一能阻止这个队列产生堆栈
    溢出的错误是在你的应用中实现 **back-presure**的代码。 *Been there,suffered that*。
*   仅针对于新的连接，时间花费在创建所有相关的 *actor*和建立TCP连接。通常创建 *actor*不会成为问题，但是建立TCP连接可能需要一会儿
    时间取决于网络情况   
        
作为最后的建议，总是根据上述这些开销总和来配置配置文件，确定对于你的应用是有意义的。如果你的应用能够忍受最长5s延时的响应，调整所有
相关联配置来保证 *Spray* 5s后不会工作在这些超时请求上。同样，我们鼓励你在开发环境中模拟这里提到的各种情况(请求数大于连接数，连接
时间慢，服务端响应慢等等)，通过你得到的结果进行调整，很多cpu周期和可能的崩溃能被这样测试挽救。         