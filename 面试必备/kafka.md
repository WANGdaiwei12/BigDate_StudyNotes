[TOC]



##### flume组成

```tex
一个flume由多个agent组成，agent由3部分组成 Source，Channel，Sink。
agent以时间（event）的方式将数据从源头送往目的地，Event 由 Header 和 Body 俩部分组成，Header 存放事件的头信息，其内部数据为 K-V 的键值对形式，Body 存放数据本体，其内部为字节（Byte）数组。
1）source：用于采集数据，Source是产生数据流的地方，同时Source会将产生的数据流传输到Channel，这个有点类似于Java IO部分的Channel
常用的有exec：命令 ；spooling directory：目录 netcat：ip：port  taildir ：断点续传 
avro：上一个flume的sink下沉的数据
2）channel：Source和Sink之间的缓冲区，类似于一个队列，常用的有memory channel，file channel，以及kafkachannel
3）sink：从Channel收集数据，将数据写到目标源(可以是下一个Source，也可以是HDFS或者HBase)。 常用的有hdfs，avro，hbase，kafka
flume的选择器：
1 ，每个通道都复制一份文件，replicating 。
2 ，选择性发往某个通道，Multiplexing 。
```

##### flume使用sink hdfs 导致小文件过多的影响和处理

```tex
元数据层面：每个小文件都有一份元数据，其中包括文件路径，文件名，所有者，所属组，权限，创建时间等，这些信息都保存在Namenode内存中。所以小文件过多，会占用Namenode服务器大量内存，影响Namenode性能和使用寿命
计算层面：默认情况下MR会对每个小文件启用一个Map任务计算，非常影响计算性能。同时也影响磁盘寻址时间。
flume 中处理：
默认的这三个参数配置写入HDFS后会产生小文件，hdfs.rollInterval、hdfs.rollSize、hdfs.rollCount,控制生成新文件的时间，大小。
(1) hdfs.rollInterval：文件创建超多少秒时会滚动生成新文件
(2) hdfs.rollSize： 文件在达到多少个字节时会滚动生成新文件
(3) hdfs.rollCount：当event个数达到多少个的时候会滚动生成新文件
```

##### kafka的组成和工作机制

```tex
Producer：即生产者，消息的产生者，是消息的入口
Broker：Broker 是 kafka 一个实例，每个服务器上有一个或多个 kafka 的实例，简单的理解就是一台 kafka 服务器，kafka cluster表示集群的意思
Topic：消息的主题，可以理解为消息队列，kafka的数据就保存在topic。在每个 broker 上都可以创建多个 topic 。

Partition：Topic的分区，每个 topic 可以有多个分区，分区的作用是做负载，提高 kafka 的吞吐量。同一个 topic 在不同的分区的数据是不重复的，partition 的表现形式就是一个一个的文件夹!
Replication：每一个分区都有多个副本，副本的作用是做备胎，主分区(Leader)会将数据同步到从分区(Follower)。当主分区(Leader)故障的时候会选择一个备胎(Follower)上位，成为 Leader。
Message：每一条发送的消息主体。
Consumer：消费者，即消息的消费方，是消息的出口。
Consumer Group：我们可以将多个消费组组成一个消费者组，在 kafka 的设计中同一个分区的数据只能被消费者组中的某一个消费者消费。同一个消费者组的消费者可以消费同一个topic的不同分区的数据，这也是为了提高kafka的吞吐量!
Zookeeper：kafka 集群依赖 zookeeper 来保存集群的的元信息，来保证系统的可用性

工作机制：
生产者生产数据，发送到指定的topic中，根据分区器将数据发送到不同分区中，当发送成功后，且leader和fllower完成数据的同步，服务端会响应一个ack给消费者，表示发送成功，
brocker的话有多个topic的，每个topic是有多个分区的，每个分区有多个副本，每个分区中的数据是有序的，都有一个offset，支持消息持久化，消费过后的数据默认保存7天
消费者有消费者组，他们会拉取自己订阅的主题的分区的数据进行消费，同一个消费者组的消费者只能同时消费不同分区的数据。
```



##### kafka的3种分区策略

```tex
原则
1.如果指明了partition的情况下，使用partition；如果
2.没有指明partition，但是有key的情况下，将key的hash值与topic的partition数进行取余得到partition值；
3.既没有指定partition，也没有key的情况下，第一次调用时随机生成一个整数（后面每次调用在这个整数上自增），将这个值与topic可用的partition数取余得到partition值，也就是常说的round-robin算法。

kafka的分区分配策略有3种，Range和 RoundRobin、Sticky分区策略
Range是默认策略。Range是对每个 Topic而言的（即一个 Topic一个 Topic分），首 
先对同一个 Topic里面的分区按照序号进行排序，并对消费者按照字母顺序进行排序。然后 
用 Partitions分区的个数除以消费者线程的总数来决定每个消费者线程消费几个分区
Range策略的缺点在于如果Topic足够多、且分区数量不能被平均分配时，会出现消费过载的情景
RoundRobin 轮询分区策略，是把所有的 partition 和所有的 consumer 都列出来，然后按照 hascode 进行排序，最后通过轮询算法来分配 partition 给到各个消费者。前提条件是：1.每个消费者订阅的主题，必须是相同的，并且同一个消费者组里面的所有消费者的消费线程相等
Sticky分区分配策略：当订阅主体相同时，初始化分区分配和RoundRobin相似， 轮询分区策略，当触发分区重分配结果中保留了上一次分配中对于消费者C0和C2的所有分配结果，并将之前脱离消费者组的消费者消费的分区轮询的分配给现有的消费者（最少的那个开始）
	订阅主体不相同时，按消费者顺序，优先分配自己订阅的主题的所以分区，当触发分区重分配，会保留了上一次分配中对于消费者C0和C2的所有分配结果，并将之前脱离消费者组的消费者消费的分区轮询的分配给现有的消费者（最少的那个开始）
	
	什么时候会触发Rebalance
	1.旦消费者加入或退出消费组，导致消费组成员列表发生变化，消费组中的所有消费者都要执行再平衡。
	2.订阅主题分区发生变化，所有消费者也都要再平衡。

```

##### kafka的ISR副本队列

```tex
ISR（In-SyncReplicas），副本同步队列。ISR中包括 Leader和 Follower。如果 Leader 
进程挂掉，会在 ISR队列中选择一个服务作为新的 Leader。有 replica.lag.max.messages 
（延迟条数）和 replica.lag.time.max.ms（延迟时间）两个参数决定一台服务是否可以加 
入 ISR副本队列，在 0.10版本移除了 replica.lag.max.messages参数，防止服务频繁的 
进去队列。 
任意一个维度超过阈值都会把 Follower剔除出 ISR，存入 OSR（Outof-SyncReplicas） 
列表，新加入的 Follower也会先存放在 OSR中
```

##### kafka的读写流程

```tex
写流程
1.连接ZK集群，从ZK中拿到对应topic的partition信息和partition的Leader的相关信息
2.连接到对应Leader对应的broker
3.将消息发送到partition的Leader上
4.ISR队列的Follower从Leader上复制数据
5.依次返回ACK
6.直到所有ISR中的数据写完成，才完成提交，整个写过程结束
读流程
1.连接ZK集群，从ZK中拿到对应topic的partition信息和partition的Leader的相关信息
2.连接到对应Leader对应的broker
3.consumer将自己保存的offset发送给Leader
4.Leader根据offset等信息定位到segment（索引文件和日志文件）
5.根据索引文件中的内容，定位到日志文件中该偏移量对应的开始位置读取相应长度的数据并返回给consumer

```

##### kafka为什么这么快？

```tex
1) Kafka本身是分布式集群，同时采用分区技术，并发度高。 
2）顺序写磁盘 
Kafka的 producer生产数据，要写入到 log文件中，写的过程是一直追加到文件末端， 
为顺序写。官网有数据表明，同样的磁盘，顺序写能到 600M/s，而随机写只有 100K/s。 
3）零复制技术
将数据直接从磁盘文件复制到网卡设备中，而不需要经由应用程序之手，避免了重复复制操作。
```

##### kafka的消费方式以及幂等性是啥？

```tex
kafka消费的种方式
1，at most onece 接收到消息之后，会先进行commit，然后在进行处理，自动提交的参数是false，特点是数据不会重复发送，可能会消息丢失
2, at least once 接收到数据之后，会先进行数据的处理，然后进行commit，自动提交参数为false，特点是是至少接受一次数据，可能会出现数据重复
3,exactly onece 将offset作为唯一ID和消息一同被处理，并且保证处理消息的原子性，自动提交为false，消息处理成功再提交，保证了消息既不会重复也不会丢失

```

##### kafka的消息是否会丢失和重复？

```tex
要确定Kafka的消息是否丢失或重复，从两个方面分析入手：消息发送和消息消费。
1、消息发送
         Kafka消息发送有两种方式：同步（sync）和异步（async），默认是同步方式，可通过producer.type属性进行配置。Kafka通过配置request.required.acks属性来确认消息的生产：
0---表示不进行消息接收是否成功的确认；
1---表示当Leader接收成功时确认；
-1---表示Leader和Follower都接收成功时确认；
综上所述，有6种消息生产的情况，下面分情况来分析消息丢失的场景：
（1）acks=0，不和Kafka集群进行消息接收确认，则当网络异常、缓冲区满了等情况时，消息可能丢失；
（2）acks=1、同步模式下，只有Leader确认接收成功后但挂掉了，副本没有同步，数据可能丢失；
2、消息消费
Kafka消息消费有两个consumer接口，Low-level API和High-level API：
Low-level API：消费者自己维护offset等值，可以实现对Kafka的完全控制；
High-level API：封装了对parition和offset的管理，使用简单；
如果使用高级接口High-level API，可能存在一个问题就是当消息消费者从集群中把消息取出来、并提交了新的消息offset值后，还没来得及消费就挂掉了，那么下次再消费时之前没消费成功的消息就“诡异”的消失了；
解决办法：
        针对消息丢失：同步模式下，确认机制设置为-1，即让消息写入Leader和Follower之后再确认消息发送成功；异步模式下，为防止缓冲区满，可以在配置文件设置不限制阻塞超时时间，当缓冲区满时让生产者一直处于阻塞状态；
        针对消息重复：将消息的唯一标识保存到外，部介质中，每次消费时判断是否处理过即可。
```

##### kafka数据重复解决

```tex
幂等性 +ack-1+事务来实现
幂等性就是  指producer无论向broker发送了多少条重复的消息，broker只会持久化一条
具体实现原理是，kafka每次发送消息会生成PID和Sequence Number，并将这两个属性一起发送给broker，broker会将PID和Sequence Number跟消息绑定一起存起来，下次如果生产者重发相同消息，broker会检查PID和Sequence Number，如果相同不会再接收。
幂等性只解决了当前的会话且当前的分区的幂等性。跨分区、会话不能实现精准一次性投递写入。
当producer重启后，broker分配的PID（producer_id）会发生变化。切换分区后，Patition也发生了变化。最终导致<PID,Patition,SeqNumber>作为主键的key也会发生变化。
2.也可以Kafka数据重复，可以再下一级：SparkStreaming、redis或者 hive中 dwd层去重， 
去重的手段：分组、按照 id开窗只取第一个值； 
```

##### kafka controller的选举和leader的选举（也是如何保证数据的一致性）

```tex
Controller是使用zookeeper选举出来的，每个broker都往zk里面写一个/controller节点，谁先写成功，谁就成为Controller。如果controller失去连接，zk上的临时节点就会消失。其它的broker通过watcher监听到Controller下线的消息后，开始选举新的Controller。

1.kafka的leader选举使用算法最贴近微软的PacificA（剖c非可）算法，先到先得、少数服从多数，从ISR中进行选举，无论ISR中哪个副本被选为新的leader，它都知道HW（每个分区的最后的offset,）之前的数据，可以保证在切换了leader后，消费者可以继续看到HW之前已经提交的数据。
2.选出了新的leader，而新的leader并不能保证已经完全同步了之前leader的所有数据，只能保证HW之前的数据是同步过的，此时所有的follower都要将数据截断到HW的位置，再和新的leader同步数据，来保证数据一致
当宕机的leader恢复，发现新的leader中的数据和自己持有的数据不一致，此时宕机的leader会将自己的数据截断到宕机之前的hw位置，然后同步新leader的数据。宕机的leader活过来也像follower一样同步数据，来保证数据的一致性。


如果ISR中没有副本，只能从OSR中选举一个作为Leader，但是OSR中副本的数据可能会存在数据丢失，在选举新的leader时，一个基本的原则是，新的leader必须拥有旧的leader commit过的所有消息
但可能会出现脑裂，（节点不能互通的时候，出现多个leader）、惊群效应（大量的watch事件被触发）。
```

##### HW,LEO是啥

```tex
HW，LEO分别是什么？
 hw每个分区的最后的offset,于标识特定的offset，消费者只能拉取到HW之前的消息
leo某个分区要写入下一条消息的offset
LSO --- Log Start Offset ，某个分区起始的offset
消息被写入leader后，follower会主动从leader拉取消息进行消息同步，但是不同副本拉取消息的效率不同，某一时刻，follower1拉取消息完成，但是follower2只拉取了消息3，此时follower1的HW为5，follower2的HW为4，那么该分区的HW取最小值4， 当follower2同步Leader完成后，follower2的HW为5，整个分区的HW为5

通过该方式，kafka集群很大程度上保证了Leader宕机后，数据的丢失
```

##### kafka中zookeeper的作用

```tex
zookeeper 是一个分布式的协调组件，早期版本的kafka用zk做meta信息存储，
记录broker的存活状态，控制器的选举，topic 的访问控制信息，zookeeper 保存了 topic 相关配置，例如 topic 列表、每个 topic 的 partition 数量、副本的位置等等。
zookeeper还记录着 ISR 的信息，而且是实时更新的，只要发现其中有成员不正常，马上移除
consumer的消费状态，group的管理以及 offset的值。考虑到zk本身的一些因素以及整个架构较大概率存在单点问题，新版本中逐渐弱化了zookeeper的作用。
0.9版本以后，新的consumer使用了kafka内部的group coordination协议，
offset由broker维护，offset信息由一个特殊的topic “ __consumer_offsets”来保存，offset以消息形式发送到该topic并保存在broker中。这样consumer提交offset时，只需连接到broker，不用访问zk，避免了zk节点更新瓶颈。
但是broker依然依赖于ZK，zookeeper 在kafka中还用来选举controller 和 检测broker是否存活等等。
```

##### zookeeper是什么

```tex
Zookeeper是一个开源的分布式协调服务，它是集群的管理者，监视者集群中各个节点的状态，根据节点提交的反馈进行下一步合理操作，最终，将简单易用的接口和性能高效、功能稳定的系统提供给用户
分布式应用程序可以基于zookeeper实现诸如数据发布/订阅、负载均衡、命名服务、分布式协调/通知、集群管理、master选举、分布式锁和分布式队列等功能
Zookeeper五大特征：
（1）顺序一致性
同一个client的请求按器发送顺序依次执行
（2）原子性
所有请求的响应结果在整个分布式集群环境中具备原子性，即要么整个集群中所有机器都成功的处理了某个请求，要么就都没有处理
（3）单一性
无论客户端连接到ZooKeeper集群中哪个服务器，每个客户端所看到的服务端模型都是一致的
（4）可靠性
一旦服务端数据的状态发送了变化，就会立即存储起来，除非此时有另一个请求对其进行了变更，否则数据一定是可靠的
（5）实时性（最终一致性）
当某个请求被成功处理后，ZooKeeper仅仅保证在一定的时间段内，客户端最终一定能从服务端上读取到最新的数据状态，即ZooKeeper保证数据的最终一致性
```

##### zookeeper工作原理

```tex
Zookeeper的核心是原子广播机制，这个机制保证了各个Server之间的同步。实现这个机制的协议叫做Zab协议。Zab协议有两种模式，它们分别是恢复模式（选主）和广播模式（同步）。当服务器启动或者在leader崩溃后，Zab就进入了恢复模式，当领导者被选举出来，且大多数Server完成了和leader的状态同步以后，恢复模式就结束了。状态同步保证了leader和Server具有相同的系统状态。
为了保证事务的顺序一致性，zookeeper采用了递增的事务id号（zxid）来标识事务。所有的提议（proposal（普弱剖z））都在被提出的时候加上了zxid。实现中zxid是一个64位的数字，它高32位是epoch用来标识leader关系是否改变，每次一个leader被选出来，它都会有一个新的epoch，标识当前属于那个leader的统治时期。低32位用于递增计数。
每个Server在工作过程中有三种状态：
LOOKING：当前Server不知道leader是谁，正在搜寻
LEADING：当前Server即为选举出来的leader
FOLLOWING：leader已经选举出来，当前Server与之同步
```

##### zookeeper监听器原理

```tex
1、监听原理详解： 
1）首先要有一个main()线程 
2）在main线程中创建Zookeeper客户端，这时就会创建两个线 
程，一个负责网络连接通信（connet），一个负责监听（listener）。 
3）通过connect线程将注册的监听事件发送给Zookeeper。 
4）在Zookeeper的注册监听器列表中将注册的监听事件添加到列表中。 
5）Zookeeper监听到有数据或路径变化，就会将这个消息发送 
给listener线程。 
6）listener线程内部调用了process()方法。 
```

##### zookeeper选举机制

```tex
1）半数机制：集群中半数以上机器存活，集群可用。所以 Zookeeper 适合安装奇数台 
服务器。
2）Zookeeper 虽然在配置文件中并没有指定 Master 和 Slave。但是，Zookeeper 工作时， 
是有一个节点为 Leader，其他则为 Follower，Leader 是通过内部的选举机制临时产生的。
3）以一个简单的例子来说明整个选举的过程
（1）服务器 1 启动，发起一次选举。服务器 1 投自己一票。此时服务器 1 票数一票， 
不够半数以上（3 票），选举无法完成，服务器 1 状态保持为 LOOKING； 
（2）服务器 2 启动，再发起一次选举。服务器 1 和 2 分别投自己一票并交换选票信息： 
此时服务器 1 发现服务器 2 的 ID 比自己目前投票推举的（服务器 1）大，更改选票为推举 服务器 2。此时服务器 1 票数 0 票，服务器 2 票数 2 票，没有半数以上结果，选举无法完成， 服务器 1，2 状态保持 LOOKING 
（3）服务器 3 启动，发起一次选举。此时服务器 1 和 2 都会更改选票为服务器 3。此 
次投票结果：服务器 1 为 0 票，服务器 2 为 0 票，服务器 3 为 3 票。此时服务器 3 的票数已 经超过半数，服务器 3 当选 Leader。服务器 1，2 更改状态为 FOLLOWING，服务器 3 更改 状态为 LEADING； 
（4）服务器 4 启动，发起一次选举。此时服务器 1，2，3 已经不是 LOOKING 状态， 
不会更改选票信息。交换选票信息结果：服务器 3 为 3 票，服务器 4 为 1 票。此时服务器 4 服从多数，更改选票信息为服务器 3，并更改状态为 FOLLOWING； 
（5）服务器 5 启动，同 4 一样当小弟。
当集群中服务器初始化启动或服务器运行期间无法和leader保持连接，就会开始进入leader选举：
 当一台机器进入leader选举流程，当前集群可能会处于以下两种状况：
（1）集群中本来已经存在一个leader，
	机器会试图选举leader时，会被告知当前服务器的leader信息，则会和leader机器建立连接，并进行状态同步
（2）不存在leader
选举leader规则：
	1.EPOCH大的直接胜出
	2.EPOCH相同，事务id大的胜出
	事务id相同，服务器id大的胜出
```

##### zookeeper读写流程

```tex
写流程
客户端连接到集群中某一个节点
客户端发送写请求
服务端连接节点，把该写请求转发给leader
leader处理写请求，一半以上的从节点也写成功，返回给客户端成功。
读流程
客户端连接到集群中某一节点
读请求，直接返回。
```

##### zookeeper的四种类型的znode

```tex
1、PERSISTENT-持久化目录节点
客户端与 zookeeper 断开连接后，该节点依旧存在
2、PERSISTENT_SEQUENTIAL-持久化顺序编号目录节点
客户端与 zookeeper 断开连接后，该节点依旧存在，只是 Zookeeper 给该节点名称进
行顺序编号
3、EPHEMERAL-临时目录节点
客户端与 zookeeper 断开连接后，该节点被删除
4、EPHEMERAL_SEQUENTIAL-临时顺序编号目录节点客户端与 zookeeper 断开连接后，
该节点被删除，只是 Zookeeper 给该节点名称进行顺序编号
```

##### CAP法则

```tex
CAP法则：强一致性、在分布式系统中的所有数据备份，在同一时刻是否同样的值
高可用性、分区容错性（系统中任意信息的丢失或失败不会影响系统的继续运作）； 
Zookeeper符合强一致性、高可用性！ 
```



