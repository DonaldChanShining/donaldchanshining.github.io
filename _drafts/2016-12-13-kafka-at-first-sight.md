1.kafka快的原因，硬盘顺序写（produce）+zero-copy(consume)。
  cluster->broker
 cluster -> topic -> partitions
从内容上分，topic下面一堆parttitions,  kv存储
从机器上分，broker
topic只是一个内容上的概念，下面有很多parttitions ，可以分布到各个机器上。
 
 
消费者端，一个consumer一个partition，各自记录offset?
如果是queue则消费者集群为一个group
 
1.硬盘顺序都写得速度高于或等于 内存随机
sequential disk access can in some cases be faster than random memory access
 
2.zero-copy  vs  memory
   common : 
    a.os read data from disk into pagecache(in kernel)
    b.app reads data from pagecache into user-space buffer
    c.app writes data back into kernel into socket buffer
    d.os copy data from socket buffer to NIC buffer

    zero-copy:
      send data from pagecache to network
     use sendFile!!!   os always  优化.  this tech
manager issue 282
 
import  kafka-manager时，Import老是不对  因为sb building tool和  ideal之间有着蛋疼的兼容性bug,这个时候进入project structure  /module ,delete all!!!!!!!!!!!!!!!!!!!!!
 
kafka 0.10.X
wget http://apache.fayea.com/kafka/0.10.0.1/kafka_2.11-0.10.0.1.tgz
zookeeper  3.4.9-release
wget http://apache.fayea.com/zookeeper/zookeeper-3.4.9/zookeeper-3.4.9.tar.gz   
备用方案  Yum 安装  按照运维  [How to]zookeeper安装文档
 
启动了zk就可以启动kafka-manager
/kafka-manager-1.3.1.6/application.home_IS_UNDEFINED/logs
 
 bin/kafka-server-start.sh -daemon config/server.properties




1.java.net.UnknownHostException
添加hosts
192.168.1.73 {hostname} localhost
 
 
kafka-server-start.sh
 export JMX_PORT="9910"  


镜像需要解决 netexception的问题
 容器外的jvm监控！！！！！！！！！！！！！！！！！


在server.properties 里还有另一个参数是解决这个问题的， advertised.host.name参数用来配置返回的host.name值，把这个参数配置为外网IP地址即可,
linux中kafka server返回的是hostname名，所以日志中会出现java.net.InetAddress.getCanonicalHostName的异常，开始以为是docker容器化的
原因导致的，因为对容器的网络之间不是很了解。后来得知是本身的一个问题，需要配置这个参数。
 
 
清理kafka步骤,
1.stop kafka client
2.stop kafka server
3.stop zookeeper
4.clear kafka data
 
kafka  offsets 存储于 zooker 当中，所以开始的环境配置处理不妥善，导致上传和拉取数据时，offests异常。。。。。。。。。。。
 
 
1.close kafka server
每个Broker上执行bin/kafka-server-stop.sh 
2.清理数据
删除路径 /data/kafka下的数据
3.清理zk
删除重装zk
/var/lib/zookeeper/version dir  rm
/opt/mesosphere 
 
要杀掉zk进程后重装
 
先通过topic分组 ，每条消息 也可以通过 k-v分组。
其中  key maybe可以用不同的parser  和 序列化  map  来实现属性的添加
 
 
剩下的要做的是  如何 控制 。。。  效率 ？分区 ？  partition以及实际应用中可能出现的场景和使用规范!!!!
 
 
1000W数据轮询写入了20个分区，1个replica,也就是40个partition
结果每个partition 50W，
同步状态下拉取数据，每次拉取完毕后会提交offsets的位置。 
感觉数据并没有顺序写入少几个分区快。。。
然后每个分区多几个replica？
例如　搞５个partition ,每个partition3个replica，20个。更高可用？》
擦。wtf。。
所以partition的使用问题！！！！！！！！！！！！！
 
 
   而kafka本身自带的 记录 offset的topic每个brokers上面分布了50个partition，why?
kafka-manager中展示也有一定的问题。。。。group->consumer、
完全体现不出consumer的作用。。
 
一个项目为一个consumer-group是正确的做法。
 
 每个consumer-group分配parttion?????
 
每个producer 写入 指定的parttition?>
 
CACACACACACACACA.
每个partition还有一个leader?,.
about partition的属性。。
 
 
熟悉configuration中各个参数的意思，，
in order to give a manual,you should be specific..
 
 
produce 100w消息，
打印Full内容 27S
打印计数器，17S
不打印，6s........
 
 
1000W 条 71S ,20partition,2replica
 
备份数量必须小于broker
5partition 2 replica
100W noack 7.8s   999176     
100W ack(1)  5.3s  100W,,,,
 
ack1 快于 0   3998832
 
1000W ack(1)   37S  
13998832
 
1000W ack(all)   52s
 
23998832
 
 
1000W ack(0)  68s 
 
43998252
 
测试相关
 
 
下面使用ack(1)
对topic中的partition做比较
1.5p 2r
37S
2.20p 2r
40s
3.50p 2r
36s
 
看起来partition数量并没有影响。。。
一个topic是一块物理区域
一个topic内的partition的分布和写入的逻辑 to know。。。
 
consumer消费的时候是从一个topic内的一个partitioni
如果一个topic有20个partition，那么是顺序平均写入这个20个pattition中
那么消费时。有一个consumer和patition的规则。。
每次只能一个consumer消费？》一个partition。。
 
那么consumer group来消费时，也是partition 轮询派给 各个分区消息。。
直接从硬盘到内核然后通过socket输出。。。顺序硬盘读写 
 
4000W
0 over:2470523 cost time:10249 ms
1 over:2882289 cost time:5381 ms
2 over:2477285 cost time:7197 ms
3 over:2890223 cost time:5763 ms
4 over:2882307 cost time:5449 ms
5 over:2470523 cost time:5434 ms
6 over:2882289 cost time:6941 ms
7 over:2470523 cost time:5847 ms
8 over:2890260 cost time:6300 ms
9 over:2882289 cost time:5611 ms
10 over:2397197 cost time:4644 ms
11 over:2812823 cost time:5835 ms
12 over:2882307 cost time:10976 ms
13 over:2882307 cost time:12616 ms
14 over:2824799 cost time:7874 ms
15 over:0 cost time:100001 ms
polling time 10000
 
 
断电之后,重启
zookeeper是rpm包安装，写入了系统服务 所以可以自启动
同时保留了kafka保存在zk上的节点信息
 
但是kafka需要重新启动，同理，kafka-manageｒ也需要重启。
 
数据不会丢失，但是ｃｏｎｓｕｍｅｒ的相关信息丢失了，也就是说　ｃｏｎｓｕｍｅｒ的ｏｆｆｓｅｔｓ在ｓｅｒｖｅｒ端丢失了。
如果客户端没有保存ＯＦＦＳＥＴＳ相关信息，则去拉信息时就会从最后开始拉取。。
 
重启后不会影响ｃｏｎｓｕｍｅｒ的信息的存储，。
 
同时ｍａｎａｇｅｒ的展示上面，ｃｏｎｓｕｍｅｒ的信息，组的信息都无法展示了。。。
ｋａｆｋａ－ｌｏｇ中存储的就是各个ｐａｒｔｉｔｉｏｎ中序列化之后的数据分为ｌｏｇ和ｉｎｄｅｘ。ｉｎｄｅｘ中存储的不知道什么鬼，。
 
重启之后新建ｔｏｐｉｃ，尝试新建ｔｏｐｉｃ之类的操作
发现还是一样。
消费者端默认会从尾端开始消费。避免重复消费的窘境？》
如果说消费者重启后，之间的也会丢失。
无法重新生成新的ｃｌｅａｎ的ｃｏｎｓｕｍｅｒ－ｇｒｏｕｐ去从头开始消费，同时，也无法在ｋａｆｋａ－Ｍａｎａｇｅｒ上展示出消费者的确切信息了。。。。。。。。。。。。。。
 
现在发现 consumer信息还是存储在offsets topic上面  应该被持久化了，，，
为何不行 需要进一步去发现
/consumers/groupid/ids/consumerid/topic
 
临时的znode 客户端断开时 去除。 可以重现一下。。。
 
丢失原因分析
1.Consumer有Type ,zk和kafka
  个人猜想kafka的offset存储在客户端
            zk的offset存储在zk上？。
 
 
运行一段时间之后 一切逻辑又可以回馈正常状态了，，，
whats wrong>.
到底offsets是存储和读取的位置在哪里 是个problems。。。。。。。。。。。。。。。。。
设置从头则 从头，latest 即 latest。。。。
 
重启kafka-manager， 保持连接的消费者信息 还在 ， 丢失连接的消费者信息没？
 
连接的信息在 是因为 客户端连接时 会不断的上报 自己的offsets.。。。。。。。。。。。。。。
但是为何重启manager 会产生这个现象？》
临时检测不到？
然后当已有offset的topic连接上之后，读取原来的数据 ，然后上报offsets时，就可以更新展示了？》》》
会产生重复消费的问题。。
kafka-manager只是拉取信息。。信息的存储，，重启kafka-manager不会对必要信息产生影响，比如offsets等
topic,broker属于固定的节点，所以也不会有问题。
Consumer-offsets。...............
 
先进行功能性开发，在进行容错性开发。
断开的Broker重新加入时。。。partitiion不能进行同步操作?>
可以但是需要去进行一个触发操作。然后partitioN会同步其他isr的partition
partition只可以加，加完后重新轮询添加数据。。
 
 
因为数据是轮询写入partition当中
而消费者也是轮询进行消费。
当消费者大于副本时，会如何配置消费方案？
也就是不按顺序了。。
随机分配partition给消费
所以要求分区数 大于消费者数量？
 
一个partition上只有一个offsets>
 
不能保证消息的有序性。。。
 
decide the size of kafka
 
kafka-benchmark
 
文件句柄数，。
 
关于record
无key的按照规则轮询分发给topic上的parttion
key一样的会分发到同一个parttition
确保key一样的情况下 完成队列的语义？》 
但是效率上会产生影响。
record上加上了5个信系  ，topic,partition,key,value,timestamp
 
partition和key可以不指定
 
partition是选择指定到哪个pattion上面
不指定轮询，。
 
Key是同Key的分配到统一patttion
不指定轮询。
 
 
record序列化时，
key value 分别和topic一起序列化
 
每个topic一块儿区域
然后每个partition一块区域,
但是统一topic中的partition都是logic上的一块物理区域上。
多个producer – 多个 consumer。。
 
当加入broker时 可以使用界面管理工具直接reassign partition数据到新的增的broker上。
More partition means more 并发性
但是意味着更多的文件句柄
但是有时候会出现Broker-skew 所以并不清楚具体原因
 
play是直接基于netty实现的request->result..
回归kafka
 
 
设计思路
加入props的概念
即在topic的下面 多一层可以提交的Props.
订阅topic时，可以同时添加属性或者All
 
普通消息队列  就是一个topic 然后，消费者端分为不同的group即可
 
就是添加一个属性标志，不用服务端来解析。
属性会序列化到broker上，最好的就是在broker上不可见，作为Body的一部分
 
对于key，同key的默认会发到同一part上
如果制订了Part则发到指定Part.
 
拉取数据时提供一个filter操作?
coordinatior maybe可以开发。。。。
 
加密设置。
好像就这么多事情。。。
 
rabbitmq中  如果有未消费掉的消息，unack   mq服务端会不推送 interesting...
 
(buffer.get() & 0xFF)   buffer read时
好像在哪里见过.
 
/**
 * The "magic" value
 * When magic value is 0, the message uses absolute offset and does not have a timestamp field.
 * When magic value is 1, the message uses relative offset and has a timestamp field.
 */
public static void writeUnsignedInt(ByteBuffer buffer, int index, long value) {
    buffer.putInt(index, (int) (value & 0xffffffffL));
}

/**
 * Write an unsigned integer in little-endian format to the {@link OutputStream}.
 *
 * @param out The stream to write to
 * @param value The value to write
 */
public static void writeUnsignedIntLE(OutputStream out, int value) throws IOException {
    out.write(value >>> 8 * 0);
 out.write(value >>> 8 * 1);
 out.write(value >>> 8 * 2);
 out.write(value >>> 8 * 3);
}
 
关于日志清理：
配置topic时，会有日志保留的情况配置，如果不配置 则使用Broker的清理配置，默认都是7天
同时，日志清理时，只清理了Log的内容，index并不会清理，UI上展示的也是parittion上给出的Index的值。
而实际上的log清理情况并没有具体的展示。。但是日志内容的清理 确实是按照规则来的。。。
 
 
关于kafka的CORE中包的重点摘要：
1.LOG
    kafka持久化到磁盘上的数据单位叫做Log
   1.1CleanerConfig
       关于清理log的参数的一个样例类。
       默认最大msg size  32M
      后台 check清理log间隔 15s
      iobuffer 1m    清除重复数据4M
   1.2 fileMsgSet
         基于硬盘的消息集.
        构造器包括一个起始位置，FILECHANNEL (nio需要使用).,file对象，是否将起始位置用于分割这一块物理区域。
        1.2.1 read方法
          读取构造的一个子集，fileMsgSet,.指定位置和大小。返回set的起始位置为msgset的start位置+postion。终止位置为fileend或者起始+size
        1.2.2searchFor方法
             params:targetOffset,startingPostion
              
           
   .1.3 offsetPostion
          offset为kafka中的逻辑说法
         position为实际位置.用于一个msgoffset的入口mapping 这样 set中其他数据位置是可以通过计算获取的。
 
kafka-manager丢失后 哪怕是重启 kafkaconsumer的信息就丢失了 ，，，如果是类型为zk类型的 因为存储在zk上，反而能拿到对应的offset信息
但是因为kafka-manager挂掉而导致 丢失 消费者信息的事情。。感觉 很差。。
 
 
kafka  exactly-once semantics
Kafka's semantics are straight-forward. When publishing a message we have a notion of the message being "committed" to the log. Once a published message is committed it will not be lost as long as one broker that replicates the partition to which this message was written remains "alive". The definition of alive as well as a description of which types of failures we attempt to handle will be described in more detail in the next section. For now let's assume a perfect, lossless broker and try to understand the guarantees to the producer and consumer. If a producer attempts to publish a message and experiences a network error it cannot be sure if this error happened before or after the message was committed. This is similar to the semantics of inserting into a database table with an autogenerated key.
kafka中这个语义就直接的多，当发布一条消息，我们要有一个 消息被提交到Log中的概念，一旦发布的消息被提交，它就不会丢失只要有一个broker上存在消息写入的partition的副本存活，存活的定义和我们尝试去处理什么样的错误一样会在下一章详解。现在我们假设一个完美的，不会丢失的broker，试着去理解
对生产者和消费者的保障。如果生产者尝试发布一条消息，遇到了网络错误，它不能确定这个错误发生在消息被提交之前或者之后。这和插入信息到数据库带着一个自动生成的key的语义一样。
These are not the strongest possible semantics for publishers. Although we cannot be sure of what happened in the case of a network error, it is possible to allow the producer to generate a sort of "primary key" that makes retrying the produce request idempotent. This feature is not trivial for a replicated system because of course it must work even (or especially) in the case of a server failure. With this feature it would suffice for the producer to retry until it receives acknowledgement of a successfully committed message at which point we would guarantee the message had been published exactly once. We hope to add this in a future Kafka version.
对于发布者来讲没有最健壮的可能性语义，尽管我们不能确定网络故障会发生什么情景，但是让生产者去生成一系列的主键来让重试生产请求拥有幂等性是可行的。这个特性对于一个复制系统(复制模型的系统？ replicated system)不重要，因为当然它必须甚至或者说特别在当服务端挂掉的场景能够工作。
带着这个特性，它会满足生产者的重试需求直到它收到一条成功提交的确认信息，这样在某种程度上我们能够保证消息已经被发布exactly once,我们希望加上这个特性在某个kafka版本
Not all use cases require such strong guarantees. For uses which are latency sensitive we allow the producer to specify the durability level it desires. If the producer specifies that it wants to wait on the message being committed this can take on the order of 10 ms. However the producer can also specify that it wants to perform the send completely asynchronously or that it wants to wait only until the leader (but not necessarily the followers) have the message.
不是所有的使用场景需要这么强的保证，对于那些对于延迟很敏感的应用场景，我们允许生产者按自己的需求指定持久化l。如果生产者指定他想等消息被提交后，这样需要消耗10ms的量级。然而，生产者也能够指定它像飙戏
发送完全异步的特性，或者说它像等直到领导者（追随者不需要）持有这个消息。
Now let's describe the semantics from the point-of-view of the consumer. All replicas have the exact same log with the same offsets. The consumer controls its position in this log. If the consumer never crashed it could just store this position in memory, but if the consumer fails and we want this topic partition to be taken over by another process the new process will need to choose an appropriate position from which to start processing. Let's say the consumer reads some messages -- it has several options for processing the messages and updating its position.
我们现在在消费者的视角描述语义。所有副本有一样的log和同样的offset，消费者控制自己的position在Log中的，如果消费者从来不会挂，他能够仅仅存储position在内存中。但是如果消费者挂了，我们希望这个topic的partition能够被一个其他的进程接管，这个新的进程需要选择合适的position，从这个postion开始进行消息处理。
我们接下来聊消费者读取（reads）了一些消息后，消费者对于处理消息和更新他的position的策略有一些选项。
1.It can read the messages, then save its position in the log, and finally process the messages. In this case there is a possibility that the consumer process crashes after saving its position but before saving the output of its message processing. In this case the process that took over processing would start at the saved position even though a few messages prior to that position had not been processed. This corresponds to "at-most-once" semantics as in the case of a consumer failure messages may not be processed.
它能够读取消息，然后保存自己的postion到log中，最后处理消息。在这种情况下，存在这样一种可能性：消费者进程故障发生在 保存它的postion到log后 却在存储处理消息的输出前。这样的场景下，接管处理消息的进程会在存储的postion上接着开始处理消息。哪怕在这个postion的先前的一些消息并没有被处理。
这种处理方式对应的语义是at-most-once最多一次，因为在这种消费者失败的情况下消息也许不会被处理。
2.It can read the messages, process the messages, and finally save its position. In this case there is a possibility that the consumer process crashes after processing messages but before saving its position. In this case when the new process takes over the first few messages it receives will already have been processed. This corresponds to the "at-least-once" semantics in the case of consumer failure. In many cases messages have a primary key and so the updates are idempotent (receiving the same message twice just overwrites a record with another copy of itself).
它能勾读取消息，然后处理消息，最后存储它的position。这种情况下存在这样一种可能性，消费者进程故障 发正在 处理完消息之后但是在保存postion之前。这样的场景下，当新的进程接管处理消息时，它收到的前几条消息都是已经被处理过的。这样的处理方式对应的语义at-least-once至少一次，在这种消费者失败的情况下。
在大多数情况下消息拥有一个主键（primary key）这样更新是幂等的，收到同样的消息两次只是复写一条这条消息本身的记录。
3.So what about exactly once semantics (i.e. the thing you actually want)? The limitation here is not actually a feature of the messaging system but rather the need to co-ordinate the consumer's position with what is actually stored as output. The classic way of achieving this would be to introduce a two-phase commit between the storage for the consumer position and the storage of the consumers output. But this can be handled more simply and generally by simply letting the consumer store its offset in the same place as its output. This is better because many of the output systems a consumer might want to write to will not support a two-phase commit. As an example of this, our Hadoop ETL that populates data in HDFS stores its offsets in HDFS with the data it reads so that it is guaranteed that either data and offsets are both updated or neither is. We follow similar patterns for many other data systems which require these stronger semantics and for which the messages do not have a primary key to allow for deduplication.
所以什么才是extractly-once 只有一次的语义。（你实际上想达到的效果）这里的限制实际上不是消息系统的特性，而是需要去协调消费者的postion和它实际上存储的作为输出。达到这个目标的经典做法是引入一个双向提交的机制在消费者postion的存储和消费者输出的存储之间。但是这能被更加平常和优雅的处理，使用
平常的让消费者在存储自己输出的地方存储自己的offset。这样更好因为消费者也许想要写入的 许多的输出系统 会不支持双向提交的机制。如下面这个例子，hadoop etl，填充数据到hdfs中。保存它的offsets在hdfs中 和它读的数据在一起，这样它能够保证 data和offsets同时被更新或同时不被更新，我们遵循类似的模式
在许多其他的需要更加健壮的语义的数据系统中，对于这样的系统，消息不需要一个主键来达到重复数据删除的目的。
 So effectively Kafka guarantees at-least-once delivery by default and allows the user to implement at most once delivery by disabling retries on the producer and committing its offset prior to processing a batch of messages. Exactly-once delivery requires co-operation with the destination storage system but Kafka provides the offset which makes implementing this straight-forward.
 所以kafka默认的有效率的保证了至少一次的分发机制，同时允许用户通过 禁止生产者重试提交 和 提交offset的操作优先于处理一批消息 来实现最多一次的分发机制。只有一次的分发机制，需要和目标存储系统协调，但是kafka提供自己的offset来使实现更加直接。     
 
认真再次翻译完之后，计划清晰了很多。
3种消息的api。
根据不同的底层api进行实现？《
 
重点是offset的存储和数据的处理关系。。。
reactive-kafka 
external offset-storage
 
 
 1.分级
 org - product - topic
三级
2.
groupid-clientid
org做拓展？
 
3.实现Authticate()
通过配置一些方式。
 
公司下面有产品和话题
产品和话题组成topic
 
userName 和 clientId一致？
然后使用password的md5?
客户端只需要进行配置即可。Jar里面进行加密通信？
clientId和userName保持一致。
谁/哪个客户端进行了调用，pwd进行加密验证》  
 
先实现功能，然后想优化！！！！！！！！！！
 
 
akka-http!!!!!!!!!!!!!!!!
http-proxy!