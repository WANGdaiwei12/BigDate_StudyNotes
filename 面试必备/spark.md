[TOC]

##### 1.	spark是什么,以及他的特点

```tex
spark是一个基于内存用于大规模数据处理的统一分析引擎。他内部的组成模块包含sparkcore，sparksql，sparkstreaming，sparkmlib，spark,
他的特点：
	1. 快 spark的计算速度是mapreduce计算速度的10~100倍
	2. 易用 MR支持一种计算模型，spark支持多种计算模型
	3. 通用 spark能够进行离线计算，交互式查询，实时计算，机器学习以及图计算
	4. 兼容性 spark支持大数据中的yran调度，支持mesos，可以处理hadoop计算的数据
```

##### spark与mr区别

```tex
两者都是用mr模型来进行并行计算:
1.spark基于内存，减少低效的磁盘交互；mr是基于磁盘的， MR 过程中会重复的读写 hdfs，造成大量的磁盘io操作者
2.spark是基于 DAG 的高效的调度算法以及高容错的缓存机制，提供了丰富rdd算子，可以进行迭代式计算，mr只有map和reduce，表达能力欠缺
3.场景：
Hadoop/MapReduce 和 Spark 最适合的都是做离线型的数据分析，但 Hadoop 特别适合是单次分析的数据量“很大”的情景，而 Spark 则适用于数据量不是很大的情景。
```

##### 2.	rdd是什么，以及他的5个特性

```tex
RDD（Resilient Distributed Dataset）叫做分布式数据集，是 Spark 中最基本的数据抽象， 它代表一个不可变、可分区、里面的元素可并行计算的集合，主要有以下特点：
1 RDD 的弹性主要体现在：
	1）自动进行内存和磁盘数据存储的切换 存储弹性。
	2）基于血统的高效容错机制 （过程弹性）
	3）Task 如果失败会自动进行特定次数的重试，默认次数是 4 次。
	4）Stage 如果失败会自动进行特定次数的重试(是否执行 shuffle )
	5）Checkpoint 和 Persist 可主动或被动触发
	6）数据调度弹性：可以将多 Stage 的任务串联或并行执行，调度引擎自动处理 Stage 的失败以及Task 的失败
	7)数据分片的高度弹性：可以根据业务的特征，动态调整数据分片的个数，提升整体的应用执行效率
2.分区：计算的时候通过compute函数得到每个分区的数据
3.依赖：RDD通过算子进行转换，新的rdd包含了从其他rdd衍生所必须的信息，rdd之间维护着这种依赖，oneToOneDependency和shuffleDependency
宽依赖：一个父RDD的partition被多个子RDD的分区所依赖，会引起shuffle
窄依赖：一个父RDD的partition最多被一个子RDD的分区
4.缓存容错：通过persist或cache方法将前面的计算结果缓存下来，默认情况下persist（）会把数据以序列化的形式缓存在JVM的对空间中，存储级别是
默认是MEMORY_ONLY，项目中一般是 
MEMORY_AND_DISK_SER_2：（将RDD的Partition序列化后的对象(每一个Partition占用一个字节数组)存储在JVM中，比直接保存原始对象来说空间利用率更高，但是在读取的时候，由于需要进行反序列化，所以会占用一定量的CPU）
他的5个特性：
A list of partitions    一个分区列表
A function for computing each split  
	对rdd做计算相当于对RDD的所有分区做计算
A list of dependencies on other RDDs
	rdd之间存在依赖关系
Optionally, a Partitioner for key-value RDDs (e.g. to say that the RDD is hash-partitioned)
	key-value类型的RDD分区器，控制分区策略和分区数
Optionally, a list of preferred locations to compute each split on (e.g. block locations for an HDFS file)
	每个分区都有一个优先位置列表
```

##### 3.	常用的算子有哪些

```tex

1.类型转换的算子
	1）map():将数据集合的数据逐条进行映射转换  可以是类型的转换，也可以是值的转换
	2）mappartitions():将待处理的数据以分区为单位发送到计算节点进行处理
	3)mapPartitionsWithIndex（）：将待处理的数据以分区为单位发送到计算节点进行处理，在处理时同时可以获取当前分区索引。
	4）flatMap():将处理的数据进行扁平化，再切分压平，每一个输入的元素可以被映射成0个或多个元素
	5）glom（）：将同一分区的数据直接转换为相同类型的内存数组进行处理，分区不变
2.聚合，排序类型的算子
	1)sortByKey：在一个(K,V)的 RDD 上调用，K 必须实现 Ordered 接口(特质)，返回一个按照 key 进行排序的
	1）groupBy():将数据根据指定的规则进行分组，分区默认不变，但数据会被打乱重新组合（shuffle），
	2）reduceByKey（）  将相同的key的数据进行value的聚合操作
	3）groupByKey():将数据源中的数据，相同的key的数据分在一个组中，形成一个对偶元族，元族的第一个元素是key，第二个元素就是相同key的value的集合
	4)aggregateByKey()()  将数据根据不同的规则进行分区内计算和分区间计算（分区内和分区间的规则可以不一样）
	5)foldByKey：当分区内计算规则和分区间计算规则相同时，aggregateByKey 就可以简化为foldByKey
	6)combineByKey：对key-value 型 rdd 进行聚集操作的聚集函数，类似于aggregate()，combineByKey()允许用户返回值的类型与输入不一致
	7) cogroup ：在类型为(K,V)和(K,W)的RDD 上调用，返回一个(K,(Iterable<V>,Iterable<W>))类型的 RDD
3.缓存类型的算子
	1）cache()：其底层调用了persist(),默认数据只能存在内存中，
	2）persist()默认持久化操作只能在内存中，如果需要可以更改保存级别
	3）checkpoint(保存路径): 需要落盘，当作业执行完后，不会删除的,一般保存路径是在分布式存储系统：hdfs
	StorageLevel.DISK_ONLY
4.调整分区的算子
	1）coalesce:根据数据量缩减分区，用于大数据集过滤后，提高小数据集的执行效率,coalesce shuffle参数为false的情况, 不会重新混洗分区, 它是合并分区
	2）repartition():减少和增大分区,存在shuffle,其底层调用的是coalesce（）,参数shuffle的默认值为true；
	3)partitionBy():将数据按照指定Partitioner 重新进行分区。Spark 默认的分区器是HashPartitioner   import org.apache.spark.HashPartitioner,还有一个范文分区器RangePartitioner，和一个私有的pythonPartitioner
    // 自定义分区器 继承partitioner抽象类，重写里面的方法
5.多个Rdd关联类型的算子
交集intersection，并集union，差集subtract  笛卡儿积cartesian
join：将两个相同key的数据组成新的数据（key,（v1,v2））
 zip (拉链):将两个 RDD 中的元素，以键值对的形式进行合并
行动算子：
	计算类型
	1）reduce（）：聚合rdd中的所以元素，先聚合分区内数据，在聚合分区间数据
	2）Collect（）：收集excouse端的数据，以数组Array的形式返回数据集所以的元素
	3）count() 数据源中数据的个数
	4）first()  获取数据源中第一个数据
	5）take(N):获取N个数据
	6）takeOrdered：数据排序后，取N个数据
	7）top  ：取前几个数据
	8) takeSample(withReplacement,num, [seed])返回一个数组，
    //  该数组由从数据集中随机采样的num个元素组成，可以选择是否用随机数替换不足的部分，seed用于指定随机数生成器种子
    9) Aggregate	 初始值参与分区内的计算，并且参与分区间的计算
    10) Fold 当分区内和分区间的计算逻辑一样
    11) CountByValue  可以统计rdd集合中每个元素出现的个数
    12) CountByKey   统计key出现的次数
    输出类型
    saveAsTextFile（），
    saveAsObjectFile（），
    wholeTextFiles()：对于大量的小文件读取效率比较高，大文件效果没有那么高。
    saveAsSequenceFile（）
```

##### 4.	rdd，dateframe和dateset之间的区别

```tex
定义方面来说：
RDD代表弹性分布式数据集，是spark最基本的数据抽象
df：与rdd类实，以命名列构成的一个分布式数据容器，除数据外，还记录着数据的结构信息schema
ds：整合了rdd和df的优点，支持结构化和非结构化的数据，既支持自定义对象，又保证了类型转换安全，
时间方面来说：
rdd spark1.0版本，
dataframe spark1.3版本，引入了schema和off-heap
dataset	spark1.6，并带来的一个新的概念Encoder，实现了自定义的序列化格式，还进行了很多优化
序列化方面来说：
RDD是分布式的 Java对象的集合序列化时使用的java的序列化
DataFrame是分布式的Row对象的集合，数据以二进制的格式存储在堆外内存中，并且schema是已知的，可以直接在内存上进行转换，避免了昂贵的java序列化
dataset在序列化时会使用自定义的数据编码器进行序列化和反序列化，
序列化数据时，Encoder产生字节码与off-heap进行交互, 能够达到按需访问数据的效果, 而不用反序列化整个对象.

spark2.X版本之后，默认使用kyro序列化

场景：当需要写一些适配性很强的函数时，行的类型不能确定，这时使用df或ds【row】很好的解决
```

##### 5.	spark的shuffle

```tex
Spark程序中的Shuffle操作主要有两个阶段，分为fhufflewrite阶段，shuffleRead阶段
是通过shuffleManage对象进行管理。Spark目前支持的ShuffleMange模式主要有两种：HashShuffleMagnage 和SortShuffleManage
hashShuffleMagnage又分优化和未经优化的
未经优化的在shufflewrite阶段，Map Task将数据的key进行分区，然后写入buffer缓冲区，待缓冲区达到阈值时开始溢写文件，，生成文件的个数=map * reduce的乘积
shuffle read阶段会一边进行数据的拉取，一边进行聚合操作，每次都会拉取和缓冲区相同大小的数据，然后通过内存中的一个map进行归并聚合操作，直到所有的数据归并结束
存在的问题，shuffle数量不大的情况下，文件个数过多，严重影响io的性能，所以引入了consolidate机制，出现shuffileFileGroup的概念，当第一批并行执行的每个task都会创建一个shuffileGrop，并会将数据写入对应磁盘文件中，下一批Task就会复用之前已有的shuffleGrop，包括其中的文件，
此时，文件的个数是core和R的乘积，文件数可以大大减少
sortshuffle：一种普通运行机制，一种bypass机制
普通的话，数据会先写入一个内存数据结构中（默5M），此时根据不同的shuffile算子，选用不同的数据结构，如果是聚合类的shuffle算子，那么会选用map数据结构，一边进行聚合，一边写入内存，如果是join这种普通的shuffle算子，那么会选用array数组结构，直接写入内存。当内存缓冲区达到阈值时，会进行溢写操作，在溢写到磁盘之前，会根据key进行排序，之后分批次写入磁盘文件中，默认batch 10000条，当task处理完所有的数据后，会产生多个临时溢写文件，会对所有的临时文件进行合并，生产一个最终的溢写文件，同时还会生成一个对应的索引文件，其中标识了下游各个task数据在文件中的start offset和end offset
shuffle read阶段则会根据索引读取每个磁盘文件的数据进行处理
bypash运行机制触发的两个条件：
	1.不是聚合类型的shuffle算子
	2.map task的数量小于bypassMergeThreshold的值，默认是200。
map task数据溢写到磁盘之前不会对数据进行排序。
和未经优化的hashshuffle一样，每个map task都会为下游的每个reduce task生成一个文件，但是最后会进行数据的归并，合并为一个磁盘文件。好处就是不对数据进行排序，节省了这一部分的资源开销

```

##### 6.	spark的背压机制

```tex
spark的背压机制：
根据JobScheduler反馈作业的执行信息来动态调整采集器采集消息的速度与执行器处理消息的速度。
设置静态配制参数“spark.streaming.receiver.maxRate”的值来实现
注意：
当数据量大，且不可控时，需要开启背压机制，动态调整接收与处理的速率。反之当数据量稳定时需要关闭背压机制，自己手动设置接收器速率，因为背压机制会一直占用计算资源进行动态调整的计算。
```

##### 7. spark作业提交参数和优化

```tex
bin/spark-submit \
--class com.xyz.bigdata.calendar.PeriodCalculator \
--master yarn \
--deploy-mode cluster \
--queue default_queue \
--num-executors 50 \
--executor-cores 2 \
--executor-memory 4G \
--driver-memory 2G \
--conf "spark.default.parallelism=250" \
--conf "spark.shuffle.memoryFraction=0.3" \
--conf "spark.storage.memoryFraction=0.5" \
--conf "spark.driver.extraJavaOptions=-XX:+UseG1GC" \
--conf "spark.executor.extraJavaOptions=-XX:+UseG1GC" \
--verbose \
```

##### 8.hadoop和spark的shuffle相同和差异

```tex
在spark1.2版本之前默认使用的hashshuffle
采用sortShuffle起到了两点好处：
1. 小文件明显变少了，一个task只生成一个file文件
2. file文件整体有序，加上索引文件的辅助，查找变快，
总体来看spark的shuffle就是基于hadoop的shuffle模型的，都是会将mapper的数据输出进行partition，
先写入缓存中，然后进行溢写，随后不同的partition送到不同的reduce里，
细节方面的话
但是mapreduce的进行combine和reduce时都会对数据进行排序，避免了合并时查询所有数据，而spark的sortshuffle才会进行排序，hashshuffle不会进行排序，想要排序，需要自己调用sortByKey() 
从实现角度来说，mr有明显的阶段划分，每个阶段负责实现每个的功能，spark的shuffle并没有明显的阶段划分，只有不同的 stage 和一系列的transformation()，也可以理解成，只分成了shuffle read和shuffle write阶段
mr的计算是基于磁盘的，会落盘，spark是基于内存的不落盘。
```



##### 9.spark的工作机制

```tex
用户在client端提交作业后，
1）构建Application的运行环境，Driver创建一个SparkContext会
2）SparkContext向资源管理器（Standalone、Mesos、Yarn）申请Executor资源，资源管理器启动StandaloneExecutorbackend（Executor）
3）Executor向SparkContext申请Task 
4）SparkContext将应用程序分发给Executor
5）SparkContext就建成DAG图，DAGScheduler将DAG图解析成Stage，每个Stage有多个task，形成taskset发送给task Scheduler，由task Scheduler将Task发送给Executor运行
6）Task在Executor上运行，运行完释放所有资源
```

##### 11 Cache(),persist,checkpoint之间的区别、reduceByKey、foldByKey、aggregateByKey、combineByKey 的区别、reduceBykey和groupbykey的区别

```tex
1》从shuffle的角度：reduceByKey和groupByKey都存在shuffle的操作，但是reduceByKey可以在
     shuffle前对分区内相同的key的数据进行聚合功能，减少落盘的数量，而groupbykey只是进行分组，不存在数据量减少的问题，所以reduceByKey性能更高
 2》从功能的角度：reduceByKey其实包含分组和聚合的功能呢，groupByKey只能分组不能聚合。
 reduceByKey对于类似于——求多个分区的最大值的和的需求，它做不到
======================================================================= 
 他们底层都调用了combineByKeyWithClassTag（）这个函数
  reduceByKey: 相同 key 的第一个数据不进行任何计算，分区内和分区间计算规则相同
  FoldByKey: 相同 key 的第一个数据和初始值进行分区内计算，分区内和分区间计算规则相同
  AggregateByKey：相同 key 的第一个数据和初始值进行分区内计算，分区内和分区间计算规则可以不相同
  CombineByKey:当计算时，发现数据结构不满足要求时，可以让第一个数据转换结构。分区内和分区间计算规则不相同。
========================================================================
  cache：将数据存储在计算节点的内存中，会在血缘关系中添加新的依赖，一旦出现问题，可以重头进行计算
  persist(StorageLevel.DISK_ONLY) :会将数据临时存储在磁盘文件中进行数据重用，涉及到磁盘io，性能较低，但数据安全，如果作业执行完毕，临时保存的数据文件就会丢失
  checkpoint：将数据长久的保存在磁盘文件中进行数据重用，涉及磁盘IO,但数据安全，
 执行过程中会切断血缘关系，重新建立起新的血缘关系。   
```

##### 10. 广播变量的5个注意  累加器

```tex

  累加器是共享可修改变量，用来对信息进行聚合，而广播变量是共享只读变量，用来高效分发较大的对象
  广播变量：闭包数据都是以Task为单位发送的，每个任务包含了大量闭包数据，这样会导致
  一个Executor里包含大量重复的数据，并且占用大量内存

1、广播变量在Driver端定义
2、广播变量在Execoutor只能读取不能修改
3、广播变量的值只能在Driver端修改
4、不能将RDD广播出去，RDD不存数据，可以将RDD的结果广播出去，rdd.collect()
5、广播变量使用BlockManager管理，广播变量一般存储在内存中(Executoion区域)
```

##### 11从kafka获取数据的两种方式有什么区别

```tex
receive方式：使用高级API实现来实现的，接收的数据存储在spark Executor的内存中，这样会增加内存使用压力，如果消费不及时，还会造成数据积压，随后由sparkstreaming启动的job去处理数据，为了确保数据零丢失，需要启动spark streaming的预写日志机制，但会造成数据的延迟加大，并且offset值是由zookeeper进行维护的，可能会出现数据重复的情况，此外，kafka的分区数与sparkstreaming生成的rdd分区无关，即使增加了某个topic的数量，并不会增加spark处理数据的并行度，但可以使用不同的消费者组，和topic创建多个输入DStream,以使用多个receive并行接收数据
direct方式：直连方式，使用的低级api来实现的，数据由kafka进行保存的，因为kakfa的副本机制，所以数据不会丢失，当需要消费的时候，会主动拉取kafak的数据，并且偏移量由自身进行管理，所以可以保证数据消费仅一次，此外spark会创建和kafkapartition一样多的rddpartition，增加分区的数量，可以增加spark处理数据的并行度
```

##### 12. spark性能调优

```tex
1.常规性能调优
	1.尽量重复使用rdd和rdd持久化
	2.调节并行度 spark.default.parallelism  500~1000   在sql中默认200
	3.使用广播变量和kyro序列化
	4.将任务分配的资源调节到可使用资源的最大限度
2.算子调优
	1.尽量使用mappartition，foreachpartition算子，一次处理一个分区，提高效率
	2.filter与1coalesce的配合使用，因为filter可能会出现数据倾斜，使用coalesce缩减分区，使每个分区的数量尽量均匀，紧凑，以便后面的计算
	3.repartition解决了sparksql低并行问题，因为并行度的设置对sparksql不生效，spark会自己默认根据表对应的hdfs文件的split个数自动设置spark，但可以将sparksql查出来的rdd立即使用repartition，重新分成多个分区，因不在设计sparksql，所以就可以手动设置并行度了  
	或者--conf spark.sql.shuffle.partitions=1000 shuffle的分区数
	4.优先使用reduceBykey本地聚合
3.shuffle调优
	1.调节map端缓冲区大小 spark。shuffle。file。buffer  32kb     64kb
	2.调节reduce端拉取数据缓冲区 spark。reduce。maxsizeInFlight 48mb 96mb
	3.调节reduce端拉取数据重试次数 spark。shuffle。io。maxRetries 3次  6次
	4.调节reduce端拉取数据等待间隔 spark。shuffle。io。retrywait 5s  60m 
	5.调节sortshuffle排序操作阈值 spark。shuffle。sort。bypassMerge。Threshold 200  400
4.jvm调优
	降低cache操作的内存占比  spark。storage.memoryfraction  0.6  0.4
	调节Executor堆外内存   --conf spark.yarn.executor.memoryOverhead =2048MB 300MB
	调节连接等待时长   --conf spark.core.connection.ack.wait.timeout=300s  60s
```

##### 13 spark数据倾斜处理

```tex
原因：task分配不均匀，shuffle是大量相同的key被分配到相同的分区中
解决：1.尽量避免shuffle过程，比如大表跟小表进行join的时候，一般需要进行shuffle将所有key打散，shuffle操作时，大量相同的key被分配到相同的分区中，，发送到reduce进行计算，在map端进行进行join，把小表的数据通过broadcast的方式发送到executor，之后直接在map 进行join计算，提高效率
spark.sql.autoBroadcastJoinThreshold是控制broadcast的阈值，默认10M，当小于10M自动broadcast join
	 2.增大key的粒度，减少数据倾斜的可能
	 3.通过repartition,spark.default.parallelism和自定义分区 来增大并行数量
	 3.提高shuffle操作的reduce并行度，大部分shuffle算子都可以传入一个并行度的设置参数
	 4.sparksql的话，可以设置spark.sql.shuffle.partitions 设置shuffle read task的并行度，尽可能缓解和减轻数据倾斜
	 5.通过sample采样，对倾斜key单独进行处理
```

##### 14 spark架构中的基本组件(Drive,executor)和四种模式

```tex
Driver:是一个进程，spark驱动器节点，主要负责将用户程序转化为作业，在Executor之间调度任务，跟踪Executor的执行情况
executor：是一个jvm进程，负责spark作业的具体任务，并将结果返回给driver进程，它们通过自身的blockManager为用户程序中需要缓存的RDD提供内存式缓存。
Master：在standalone模式中即为主节点，控制整个集群，监控worker。在YARN模式中为资源管理器
Worker：从节点，负责控制计算节点，启动Executor或者Driver。在YARN模式中为NodeManager，负责计算节点的控制。
四种模式：
1.本地模式:spark单击运行，主要用于测试
2.standalong模式：构建一个master和多个slave构成的spark集群，资源管理器和任务监控由spark自身监控，适用于数据量不多的情况
3.spark on yarn模式：资源管理和任务监控由yarn管理，不需要构建spark集群，适用于你的集群中同时运行hadoop和spark，兼容性更好
4.spark on mesos模式：资源管理和任务监控由yarn管理，不需要构建spark集群，适合你的集群不仅运行了spark和hadoop，还运行了docker，此时mesos更通用
```

##### 15 spark为什么是赖加载的，

```tex
好处：只有当真正需要资源的时候，再去加载，节省了内存资源
```

##### Spark on Mesos中，什么是的粗粒度分配，什么是细粒度分配，各自的优点和缺点是什么？

```tex
1）粗粒度：启动时就分配好资源， 程序启动，后续具体使用就使用分配好的资源，不需要再分配资源；好处：作业特别多时，资源复用率高，适合粗粒度；不好：容易资源浪费，假如一个job有1000个task，完成了999个，还有一个没完成，那么使用粗粒度，999个资源就会闲置在那里，资源浪费。
2）细粒度分配：用资源的时候分配，用完了就立即回收资源，启动会麻烦一点，启动一次分配一次，会比较麻烦。
```



























