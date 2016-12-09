---
layout: default
title: 2016-09-27 Spray学习总结
---

## spray是什么  
spray是一个基于akka的轻量级的scala库，并提供服务端和客户端的REST形式的HTTP支持。

#### 设计原则  
1.全异步，无锁  
2.基于Akka的Actor模型和Future  
3.高性能（特别体现在Spray低层次的组件设计上，可以在高负载的情况下高性能的运行）  
4.轻量级  
5.模块化，松耦合  
   
#### 使用背景  
笔者陆陆续续使用spray一年的过程中，主要是基于*spray-routing*和*spray-can*搭建应用为其他应用提供rest服务，
使用过程中因为忽视了*spray-can*的配置出现过一些问题，所以必须理解*spray*中一些配置的含义，才能够更好更高效的
使用和进行性能调优。本文是基于*spray*的官方文档，*io.kamon*的一篇介绍文章，以及个人的使用体会攥写而成。本文会着重介
绍*spray-routing*的相关实现机制,*spray-can*的客户端的配置和*spray*内部实现对于配置的相关处理方式，理解相关的概念对*akka-http*的使用也有一定的帮助。  

## 小议spray-routing
先结合一段代码来看，*spray-routing*提供服务的形式：
    
    object Main extends App with SimpleRoutingApp {
    
      implicit val system = ActorSystem("my-system")
    
      startServer(interface = "localhost", port = 8080) {
        path("hello") {
          get {
            complete {
              <h1>Say hello to spray</h1>
            }
          }
        }
      }
    }
    
可以看出，*spray*以一种比较简洁直观的方式来提供服务，熟练之后，可以非常高效的进行相关开发但是需要对`spray`自身的DSL有一定的了解。  
这里必须提及到两个概念：*Route*和 *Directives*。

#### Route
我们先看代码中对*Route*的定义：
  
    type Route = RequestContext => Unit
    
可以看作一个方法类型的别名：接收 *RequestContext*作为参数，返回*Unit*.表面上看*Route*什么都不返回。  
实际上我们看看*RequestContext*的定义和说明：

    /**
     * Immutable object encapsulating the context of an [[spray.http.HttpRequest]]
     * as it flows through a ''spray'' Route structure.
     */
    case class RequestContext(request: HttpRequest, responder: ActorRef, unmatchedPath: Uri.Path) {
    ...
    }
    
当一个请求在*spray*的路由中流动（flow）时，包含了http请求及其在*route*中的上下文的不可变对象,从构造器的参数中也可以看出。  
所有的请求会以一种连续性的形式（*continuation-style*）处理,通过*RequestContext*中的*responder*来实现，*responder*即
响应该请求的actor引用.这种设计模式下我们可以以一种发后即忘（*fire-and-forget*）的方式发送*RequestContext*到另外的*Actor*,
责任链向下传递，而不用担心如何处理发送后的各种结果。  
当一个*Route*接收到一个*RequestContext*时，能进行下面三种操作之一：  
1. 通过调用`requestContext.complete{....}`来完成这次请求，即使用*complete*方法返回一个*response*给调用的客户端作为对本次请求的响应
2. 通过调用`requestContext.reject{....}`来拒绝这次请求，即使用*reject*方法表明这条*route*不接受本次请求
3. 忽略这个请求，通常是由于错误导致的，客户端只能等到超时之后才会获取响应  
所以，一般设计的正常工作的*route*，当请求进入时需要以*complete*或者*reject*结束一个请求的处理。    
我们下面看看一个相对复杂的*route*的构成： 
    
    
    val route =
      a {
        b {
          c {
            ... // route 1
          } ~
          d {
            ... // route 2
          } ~
          ... // route 3
        } ~
        e {
          ... // route 4
        }
      }
      
      
*~*为*spray*自定义的连接*route*的操作符。
可以看出，一个整体的*route*由若干个自定义的小*route*构成，请求在之间至上而下流动，直到被`complete`。而*route*之间的a,b,c,d,e（包括上例中的定义请求方的*get*）
都可以看作`Directive0`，实际使用中既可以是预定义的形式，也可以是自定义的形式，请求在传递过程中，通过`Directive0`可以进行权限过滤，转换等操作，来实现业务的功能。  
     
#### Directive
同上，我们先看看`Directive0`的代码定义：
    
      type Directive0 = Directive[HNil]

`Directive`的代码定义：
      
      abstract class Directive[L <: HList] { self ⇒
        def happly(f: L ⇒ Route): Route 
        ...
      }  
      
`HList`的代码定义：

      /**  `HList` ADT base trait */
      sealed trait HList
            
`Directive`是一个需要实现`happly`方法的抽象类，`happly`是一个接受协变于`HList`的类型作为参数，返回Route类型的方法。这也可以解释上文`route`的构成中
a,b这样的可以看作`Directive0`。  
官方文档中对`Directive`的描述：`Directives`是可以用来构造任意复杂的*route*结构的建筑单元。下面来解剖一个`Directive`：
       
       name(arguments) { extractions =>
         ... // inner Route
       }

可以看出，一个`Directive`有一个命名，零个或者若干个参数，内部可能会包含一个内层*Route*。此外，`directives`能够提取一些值并使它们对于内层的*Route*透明可用，
并使这些值作为内层*Route*的方法参数。  
一个`Directive`能完成以下功能的一项或多项：
1. 在将输入的*RequestContext*传入内层的*Route*之前，将其进行转换
2. 通过自定义的业务逻辑过滤掉不需要的*RequestContext*
3. 从*RequestContext*中提取值，并使它们以`extractions`的形式被内层的*Route*使用
4. 完成请求   
其中第一点需要额外讨论一下，*RequestContext*是传递在*route*结构中的核心对象，当然也可以传递在*actors*之间，拥有不可变，轻量级，能被快速复制的特性。
当`directive`接收一个*RequestContext*对象的实例时，它能决定是不做改变的传递这个实例到内层的*Route*或者是新建一个*RequestContext*的实例的副本，对副本进行一些改变，
然后传递这个副本到内层的*Route*。这样做会对两件事情有帮助：  
1. 转换*HttpRequest*的实例
2. 添加另外一个转换`response`的方法的挂钩（`hook in`）到响应链（`responder chain`）中  
为了进一步理解`responder chain`的流程，看看`complete`的方法调用时发生了什么，以下述代码做解析：
    
    foo {
      bar {
        baz {
          ctx => ctx.complete("Hello")
        }
      }
    }
    
    
假设*foo*和*baz*的钩住响应转换的逻辑，而*bar*留下*RequestContext*的*responder*，它在传递不变的*RequestContext*到内层的*route*之前。下面
解释当`complete("Hello")`调用时发生了什么：
1. `complete`会方法创建一个`HttpResponse`并将其发送给`RequestContext`中的响应者（即`responder`）
2. 运行`baz`提供的响应转化的逻辑，并发送运行结果到`baz`接收到的`RequestContext`中的响应者(`responder`) 
3. 运行`foo`提供的响应转化的逻辑，并发送运行结果到`foo`接收到的`RequestContext`中的响应者(`responder`)
4. 原始的`RequestContext`的响应者（`responder`），即发送`http`请求的`ActorRef`，接收到响应并将它返回给客户端   

如你所见，所有的响应的处理逻辑合成了一个*directive*能选择挂钩在何处的逻辑处理链。就像一个处理的流水线，没有阻塞，一个请求经过所有的*route*操作后，
返回*response*给请求方，每一步都不需要等待，相当于责任链通过消息一直在传递，每一个*directive*专注于对收到的*RequestContext*进行*加工*,不需要关心
下一步，只需要加工完毕后返回给*RequestContext*中的*responder*即可，这样极大程度的提高了对请求的处理速度，相比于一般的Http服务框架而言。
     
            
## 小议spray-can
spray-can模块是基于spray-io开发的低层次，低开销，高性能的http服务端和客户端。服务端和客户端都是纯异步，无锁，完全用scala编写（基于Akka）.
因为API的核心都围绕着Akka的抽象类型编写（如Actor和Future），所以spray-can可以很容易的集成到我们基于Akka的应用当中。
#### Http服务端
spray-can的http服务端是嵌入式的，基于actor模型，完全异步的，低层次低开销高性能的HTTP/1.1的服务端，通过Akka-io/spray-io来实现。  
服务端的作用域瞄准在必要的高效的HTTP/1.1的服务端。可以拥有以下特性：  
a.管理连接  
b.消息解析和头信息隔离  
c.超时管理（包括请求的超时和连接的超时）  
d.响应顺序（为了透明的支持pipelining）  
spray-can并没有经典http服务端的核心特性（如请求路由，文件服务，压缩等），这些特性会在更高层次的包中实现。除了这些通常的考虑，
这种设计能够保持服务端轻量级，同时能被简单的理解和掌握。这也使spray-can服务端成为一个完美的web容器来容纳使用spray-routing的应用，
因为它们两者能够很好的互补和使用彼此的接口。 
#### Http客户端
spray-can的客户端实现上拥有着和服务端同样的优势（无锁，纯异步orz）。此外提供了三种不同层次的抽象供用户使用，抽象层次从低到高： 
   
##### Connection-level  
用户完全掌握http连接的打开/关闭和期间进入的请求如何传递，处理，由哪个连接返回响应。提供了最高的灵活性，同时也带来了最低的便利性。 
   
##### Host-level  
spray-can管理特定host的连接池    
 
##### Request-level  
spray-can接管所有连接的管理     


#### spray-can配置说明

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

#### 使用Request-Level API

以一段普通的示例代码开始分析

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

结果没什么异常，只是按照我们所设想的去执行了。要注意我们对于*Request-Level*的Api设置了一个较高的超时时间（60s）,主要是为了避免在
这个测试中阻塞我们，同时我们等下会重新访问。下面让我们开始令人惊讶的测试，如果我们使用默认配置的情况下一次性发送6个请求会发生什么？
下面是修改后的代码：

    for(r <- 1 to 6) {
        val startTimestamp = System.currentTimeMillis()
        val response = clientPipeline {
          Get("http://127.0.0.1:9090/users/kamonteam/timeline")
        }
        response.onComplete(_ => println(s"Request [$r] completed in ${System.currentTimeMillis() - startTimestamp} millis."))
      }

生成了如下输出(输出顺序可能不同,当你自己测试的时候，但是最重要的部分——*响应时间的分布*还是保持相同)：
      
      Request [3] completed in 15887 millis.
      Request [4] completed in 15887 millis.
      Request [1] completed in 16033 millis.
      Request [2] completed in 15887 millis.
      Request [6] completed in 30891 millis.
      Request [5] completed in 30891 millis.

现在可能在脑海中有两个疑问：首先，为何最开始的4个请求在15s左右完成，但是最后2个请求需要30s去完成？这个问题解释很简单，*SprayClient*
对于每一个你想发送请求的host创建一个 *HttpHostConnector*,你可以把配置文件中的*host-connector*当作一个连接池，拥
有*spray.can.host-connector.max-connections*配置的连接数，给予目标host默认最大连接数为4，所以最开始的4个请求立即发送出去，其余2个请求需要
等待可用的连接。如果将*max-connections*设置为一个更高的值，这样就会有足够的连接数来立刻分发所有的请求，所有的请求都会按我们的预期
在15s左右完成。  

第二个问题是，为何一个请求能够耗时30s去完成，而在*spray.can.client.request-timeout*是默认被设置为20s的情况下不会引起超时？关于这个问题,配置文件中的描述已经说明了，
**这里的计时器不会开始计时直到连接处于正在接收响应的状态，也就是说计时可能始于后端应用接收请求后一段时间**，但是这里的
**一段时间**究竟是多长？好吧，这依赖于*SprayClient*工作引擎下的一些条件，我们通过对下面的图进行分析来看看究竟发生了什么。    
![Alt text](http://donaldhome.com/images/spray.jpg)

下面描述一下上图中的流程：  
   
1.应用代码使用一个*sendReceive*来发送一个Http请求。默认情况下，*sendReceive*会使用 *HttpManager*作为一个Http请求的中继站(即当你
在*actor-system*中获取的*IO(Http)*的对象)。 *HttpManager*是由*spray*框架控制的*actor*的根路径，同时包括服务端和客户端，但是为了简洁，我们只在
这里说明客户端相关的部分。   
2. *HttpManager*会使用在http请求中的目标host(其中包含其他细节，不作赘述)去决定http请求会发送到哪个 *HttpHostConnector*。如果对于目标host没有可用的
*HttpHostConnector*，则会创建一个，同时请求会被分发到这个 *HttpHostConnector*.此外，基于客户端连接设置(*user-agent header*, 
*idle timeout*, *request timeout*等等), *HttpManager*会找到或者创建一个被 *HttpHostConnector*所需要的 *HttpClientSettingsGroup*
来进行工作。    
3. *HttpHostConnector*使用了它的魔力。就像我们前面提到的， *HttpHostConnector*工作相当于一个连接池，它不得不找到或者创建一个合适大小的
*HttpHostConnectionSlot*来分发进来的http请求，如果不能找到， *HttpHostConnector*会将请求排队直到有一个合适的 *HttpHostConnectionSlot*
变成可用状态。根据*HTTP-pipelining*功能是否开启，请求分发的逻辑会不同。我们这里假设为默认设置`spray.can.host-connector.pipelining = off`,
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

#### 关于Host Level API        

*Host-level API*直接和 *HttpHostConnector*进行会话而不是通过 *HttpManager*来完成。在这之后，所有步骤都和我们上面描述的一致，这里的要点
是你需要发送一条`Http.HostConnectorSetup`的消息给 *HttpManager*,然后它会回复 `Http.HostConnectorInfo`包含了 *HttpHostConnector*的
引用( *ActorRef*),这样你可以通过引用直接发送请求给 *HttpHostConnector*。   

如果你决定建立和直接使用 *HttpHostConnector*，一定记住它有一个闲置超时时间(`idle-timeout`)，超过了这个时间它会被停止，它的引用也将失效，
你可以选择将`idle-timeout`设置为无限或者 *watch*这个引用并优雅的处理这些场景。我们倾向于避免使用 *Host-level API*，因为使用它
会增加一些复杂度(那些已经被 *HttpManager*所解决的问题)，此外，在绝大多数的应用场景中，自行保留处理一条 *actor message*和所有需要处理的任务
耗时相比不会带来显著的性能提升。当然，每个适用场景有它自己的需求，只有通过测试你自己的app你才会知道。

#### sendReceive Timeout的意义

看一下*sendReceive*的定义：

    type SendReceive = HttpRequest ⇒ Future[HttpResponse]
    
      def sendReceive(implicit refFactory: ActorRefFactory, executionContext: ExecutionContext,
        futureTimeout: Timeout = 60.seconds): SendReceive
    
      def sendReceive(transport: ActorRef)(implicit ec: ExecutionContext, futureTimeout: Timeout): SendReceive
      
      
在我们的例子中提供的隐式的超时是*requestTimeout*满足*sendReceive*方法的隐式参数*futureTimeout*，但是这个超时时间和所有的我们前面
提到的超时时间没有关系，它只是为了满足 *actor*的*ask*方法所需要的*timeout*参数，返回一个*Future[HttpResponse]*， *future*会被
填充当响应到达时。如果*future*的超时时间到了而没有收到响应，则*future*会被填充为*AskTimeoutException*.但是请求会继续执行，直到有一个
结果返回。

#### "继续执行"的意思

*Spray Client*并没有正在执行的请求的超时的概念，它只知道请求需要经过一系列的步骤去完成(不管成功或者失败,如同上文route中描述的一样，一个责任链的传递)，
只有经过所有步骤完成后，它才会停止执行请求并返回一个响应。如果我们修改前面例子中的*requestTimeout*变为10s，所有响应的 *future*会在10s后完成为
*AskTimeoutException*，但是， *Spray Client*依然会将所有的请求发送到我们的服务，哪怕最后2个请求需要等待15s后连接变为可用状态。修改
过后生成的输出会和下面类似：
    
    Request [1] completed in 10161 milliseconds.
    Request [2] completed in 10020 milliseconds.
    Request [3] completed in 10020 milliseconds.
    Request [4] completed in 10020 milliseconds.
    Request [5] completed in 10020 milliseconds.
    Request [6] completed in 10020 milliseconds.
    
但是大约5s过后，前4个发出的请求的对应的*response*会被收到，然后进入*dead letters*,因为没有*actor*在等待它们（上层的actor认为请求已经超时，
调用已经结束，而实际上请求是继续执行的，返回的响应被判定为无用的消息），大约再等15s后，最后2个响应到达，一样也会进入*dead letters*,输出日志会和下面类似：
    
    [06:00:12.153] [akka://spray-client-example/deadLetters] Message [spray.http.HttpResponse] from ...
    [06:00:12.153] [akka://spray-client-example/deadLetters] Message [spray.http.HttpResponse] from ...
    [06:00:12.154] [akka://spray-client-example/deadLetters] Message [spray.http.HttpResponse] from ...
    [06:00:12.154] [akka://spray-client-example/deadLetters] Message [spray.http.HttpResponse] from ...
    [06:00:27.161] [akka://spray-client-example/deadLetters] Message [spray.http.HttpResponse] from ...
    [06:00:27.161] [akka://spray-client-example/deadLetters] Message [spray.http.HttpResponse] from ...
    
#### 重试机制

最后一件你需要记住的事情是 *Spray Client*会默认重试幂等请求*spray.can.host-connector.max-retries*指定的次数，同时重试也是前面
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
        
作为最后的建议，总是根据上述这些开销总和来配置配置文件，确定这些对于应用的开发是很有意义的。如果你的应用能够忍受最长5s延时的响应，调整所有
相关联配置来保证 *Spray* 5s后不会工作在这些超时请求上。与此同时，开发时需要在开发环境中模拟这里提到的各种情况(请求数大于连接数，连接
时间慢，服务端响应慢等等)，通过得到的结果对应用进行相应调整，很多应用运行过程中可能出现的问题能被这样的测试弥补。

## 后记
总的来说，*spray*中一些设计思路真的是令人不得不佩服，充分发挥了*Akka*的优势，通过消息解耦，请求在传递和处理的过程中，
通过责任链的传递来实现处理过程的全异步，同时,DSL的设计在现在看来也是十分精妙的，*spray-route*的设计非常易于拓展和理解,使用者既可以在原生的*directive*上进行拓展，
也可以自定义各种*directive*完美融入整个*route*中。而*spray-can*中也延续了责任链传递处理这一思路，因为*actor*之间的消息交互和处理本身就拥有这样的特性，
但是切记理解*spray*的配置文件各项参数,根据自己的应用进行相关的调优和配置,来实现资源的最大化利用。