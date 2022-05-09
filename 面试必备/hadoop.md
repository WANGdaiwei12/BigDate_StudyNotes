[TOC]

##### 1.shuffle过程及优化

```tex
Shuffle就是洗牌，打乱的意思，
Map方法之后reduce方法之前这段处理过程叫shuffle
首先 map task 从split中读取数据，进行处理后，输出key/value，对键值对进行Partitioner后，存入到缓存中，缓存默认大小是100M，当缓存内容达到80M时，启动溢写操作，把缓存区数据写入一个溢写文件，在写入文件之前，会对键值对进行分区排序和合并（如果设置的话）,当该map task处理完所有数据后，需要对该map生成的所有溢写文件进行merger操作，生成一个文件作为该maptask的成果，reduce task接受到通知后，就回拉取各个map task的成果数据，放到缓存区中，当缓存区内容达到阀值时，同样执行溢写操作，生成溢写文件，当把所有的map task的成果数据读取完毕后，会把生成的所有溢写文件进行merge操作，生成一个文件作为reduce task的输出数据。

优化：
 1）数据输入阶段=======
 合并小文件：1.采用 har归档方式，将小文件归档（处理的是文件已经上传到了 HDFS上了） 
    hadoop archive -archiveName fruit.har -p / input_fruit/ directory/ -r 1  /input_fruit/
   2. 采用 CombineTextInputFormat 
    驱动类中设置：
     job.setInputFormatClass(CombineTextInputFormat.class)
     CombineTextInputFormat.setMaxInputSplitSize(job,4194304)
    3.设置jvm重用 mapreduce.job.jvm.numtasks=15
1）Map阶段 
 （1）增大环形缓冲区大小。由 100m扩大到 200m 
 （2）增大环形缓冲区溢写的比例。由 80%扩大到 90% 
 （3）减少对溢写文件的 merge次数。（10个文件，一次 20个 merge） 
 （4）不影响实际业务的前提下，采用 Combiner提前合并，减少 I/O。 
2）Reduce阶段 
 （1）合理设置 Map和 Reduce数：两个都不能设置太少，也不能设置太多。太少，会 导致 Task等待，延长处理时间；太多，会导致 Map、Reduce任务间竞争资源，造成处理 超时等错误。 
 （2）设置 Map、Reduce共存：调整 slowstart.completedmaps参数，使 Map运行 到一定程度后，Reduce也开始运行，减少 Reduce的等待时间。 
 （3）规避使用 Reduce，因为 Reduce在用于连接数据集的时候将会产生大量的网络消 耗。
 （4）增加每个 Reduce去 Map中拿数据的并行数 
3）IO传输 
  采用数据压缩的方式，减少网络 IO的的时间。安装 Snappy和 LZOP压缩编码器。
  采用sequenceFile二进制文件
  压缩： 
   （1）map输入端主要考虑数据量大小和切片，支持切片的有 Bzip2、LZO。注意：LZO 要想支持切片必须创建索引； 
   （2）map输出端主要考虑速度，速度快的 snappy、LZO； 
   （3）reduce输出端主要看具体需求，例如作为下一个 mr输入需要考虑切片，永久保 存考虑压缩率比较大的 gzip。 
 4）整体  
 	（1）mapreduce.map.memory.mb：控制分配给 MapTask内存上限，如果超过会 kill 掉 进 程，默认内存大小为 1G， 如果数据量是 128m，正常不需要调整内存；如果数据量大于 128m，可以增加 MapTask内 存，最大可以增加到 4-5g。 
 	（2）mapreduce.reduce.memory.mb：控制分配给 ReduceTask内存上限。默认内 存大小为 1G，如果数据量是 128m，正常不需要调整内存；如果数据量大于 128m，可以增加 ReduceTask内存大小为 4-5g。
 	（3）可以增加 MapTask的 CPU核数，增加 ReduceTask的 CPU核数 
  	（4）增加每个 Container的 CPU核数和内存大小 

```

##### 2.hdfs读写流程,以及保证数据的完整性

```tex
写：
1.客户端通过Distributed FileSystem模块向NameNode请求上传文件
2.NameNode检查是否已存在文件、检查权限。若通过检查，直接先将操作写入EditLog，并返回输出流对象。
3.client端按128MB的块切分文件，请求namenode第一个 Block上传到哪几个DataNode服务器上。
4.NameNode返回3个DataNode节点，分别为dn1、dn2、dn3。
5.客户端通过FSDataOutputStream模块请求dn1上传数据，dn1收到请求会一边进行副本拷贝，一边继续调用dn2，然后dn2调用dn3，将这个通信管道建立完成。
6.dn1、dn2、dn3逐级应答客户端
7.客户端开始往dn1上传第一个Block（先从磁盘读取数据放到一个本地内存缓存），以Packet（64K）为单位，dn1收到一个Packet就会传给dn2，dn2传给dn3；dn1每传一个packet会放入一个应答队列（并将该packet缓存起来），等待应答，应答成功，会删除该packet,目的是为数据丢失作一个备份。
8.当一个Block传输完成之后，客户端再次请求NameNode上传第二个Block的服务器。
9.写完数据，关闭输输出流，发送完成信号给NameNode。

2.HDFS的读流程
（1）客户端通过DistributedFileSystem向NameNode请求下载文件，NameNode通过查询元数据，找到文件块所在的DataNode地址，然后返回给客户端对象。
（2）客户端创建一个文件流，挑选一台DataNode（默认采取就近原则（负载能力达到极限会随机切换到另外的DataNode）服务器，请求读取数据。
（3）DataNode开始传输数据给客户端（从磁盘里面读取数据输入流，以Packet为单位来做校验）。
（4）客户端以串行读的方式来进行读取block(block1，读取完毕之后，再去读取block2)，并且客户端以Packet为单位接收，先在本地缓存，然后写入目标文件。

读写过程，数据完整性如何保持？
Hdfs采用的crc(32)的校验算法
通过文件校验和可以做到
	Hdfs最终存放的是我们分好的block块，而block最底层的就是 chunk，一个个chunk构成packet，一个个packet最终形成block，每个chunk(创可——数据块)中都有一个校验位。
	HDFS的client端实现了对HDFS文件内容的校验和(checksum)检查，当客户端创建一个新的HDFS文件的时候，分块会计算这个文件每个数据块的校验和，这个校验和会以隐藏文件的形式保存在同一个HDFS命名空间下，当client端从HDFS中读取文件内容后，它会检查分块的时候计算出的校验和(隐藏文件里)和读取到的文件中检验和是否匹配如果不匹配 客户端可以选择从其他的Datanade获取该数据块的副本


```

##### 3.mr工作原理

```tex
client：负责提交MR作业
Resource Manager:负责协调集群上计算资源的分配 。
Node Manager:负责启动和监视集群中机器上的计算容器（Container）
Application master:负责向RM申请资源，请求NM启动Container，并告诉Container做什么事情
Container:封装了内存、cpu、等资源
客户端提叫作业任务后，job根据作业任务获取文件信息，由InputFormat将文件按照设定的切片大小进行切片操作，并将切片的数据读入并生成一个MapTask任务；
maptask又分成5个阶段
（1）Read阶段：Map Task通过用户编写的RecordReader，从输入InputSplit中解析出一个个key/value。
（2）Map阶段：将解析出的key/value交给用户编写map()函数处理，并产生一系列新的key/value。
（3）Collect收集阶段：会调用OutputCollector.collect()收集结果，并会进行一个分区，并写入一个环形内存缓冲区中。
（4）Spill阶段：当环形缓冲区满后，会将数据写到本地磁盘上，生成一个溢写文件，在数据写入本地磁盘之前，会对数据进行一次本地排序，合并、压缩等操作，合并，压缩是可选的。
（5）Combine阶段：当所有数据处理完成后，MapTask对所有临时文件进行一次合并，以确保最终只会生成一个数据文件。
reduce阶段启动多个reducetask，每个reduce分为4个阶段
（1）Copy阶段：ReduceTask从各个MapTask上拉取自己对应分区的数据，写入到缓存区，如果缓存区达到阈值，溢写到磁盘上，生产溢写文件。
（2）Merge阶段：在远程拷贝数据的同时，还会对生成的溢写文件进行合并，以防止内存使用过多或磁盘上文件过多。
（3）Sort阶段：ReduceTask只需对所有数据进行一次归并排序即可。因为各个MapTask已经实现对自己的处理结果进行了局部排序，
（4）Reduce阶段：reduce()函数将计算结果输出。
（5）write阶段：outputFormat从Reduce中获取数据并通过RecordWriter将数据写入文件
```

##### 4.yarn工作机制

```tex
1. Mr 程序会提交到客户端所在的节点，如果是（集群模式创建了 YarnRunner，本地模式是LocalRunner）
2.YarnRunner 向 RM 申请一个Application
3. application启动后会将运行所需要的资源（Job.split，Job.xml，wc.jar）提交到
指定的hdfs资源路径上，当资源提交完毕，申请运行 mrAppMaster
RM 将用户的请求初始化成一个Task，并将其放入调度队列中，其中一个 NM 领取到Task任务，会创建容器container，运行mrAppMaster，并下载 job 资源到本地，
随后MRAppmaster向RM申请运行 MapTask 资源（根据切片信息申请开启的 MapTask 数量）
rm将运行的maptask任务分配给另外两个NodeManager，分别领取任务并创建容器，
接着，mr向两个接受到任务的nodemanager发送程序启动的脚本，这两个nodemanager分别启动
maptask对数据进行分区，排序，
MrAppMaster 等待所有的mapTask运行完毕后，向RM申请容器，运行reducetask，reducetask启动后向maptask获取相应分区的数据进行处理，处理完成后，MR向RM申请注销自己请释放资源
```

##### 6.hadoop数据倾斜的处理

```tex
1）提前在map进行combine，减少传输的数据量
在Mapper加上combiner相当于提前进行reduce，即把一个Mapper中的相同key进行了聚合，减少shuffle过程中传输的数据量，以及Reducer端的计算量。

如果导致数据倾斜的key 大量分布在不同的mapper的时候，这种方法就不是很有效了

2）数据倾斜的key 大量分布在不同的mapper

在这种情况，大致有如下几种方法：

 1.「局部聚合加全局聚合」
	第一次在map阶段对那些导致了数据倾斜的key 加上1到n的随机前缀，这样本来相同的key 也会被分到多个Reducer 中进行局部聚合，数量就会大大降低。
	第二次mapreduce，去掉key的随机前缀，进行全局聚合。
	「思想」：二次mr，第一次将key随机散列到不同 reducer 进行处理达到负载均衡目的。第二次再根据去掉key的随机前缀，按原key进行reduce处理。

2.「增加Reducer，提升并行度」，减少了反生数据倾斜的可能
    JobConf.setNumReduceTasks(int)
3.「实现自定义分区」
根据数据分布情况，自定义分区，将key均匀分配到不同Reducer
```

hadoop调度器总结

```tex
（1）FIFO队列调度器：
创建一个任务队列，它先按照作业的优先级高低，再按照到达时间的先后选择被执行的作业。
（2）计算能力调度器Capacity Scheduler
支持多个队列，多个队列可以同时执行，每个队列可配置一定的资源量，每个队列采用FIFO调度策略，hadoop2.7默认
（3）公平调度器Fair Scheduler 支持多队列多用户，第一个程序启动是，可以占用其他资源的队列，当其他队列有任务要提交时，占用资源的队列需要将需要的资源还给该任务。还资源的时候比较慢。 cdh默认
```

##### hadoop正常启动有哪些进程

```tex
1) NameNode它是hadoop中的主服务器，管理文件系统名称空间和对集群中存储的文件的访
问，保存有metadate。
2）SecondaryNameNode它不是namenode的冗余守护进程，而是提供周期检查点和清理任务。
帮助NN合并editslog，减少NN启动时间。
3）DataNode它负责管理连接到节点的存储（一个集群中可以有多个节点）。每个存储数据的
节点运行一个datanode守护进程。
4）ResourceManager（JobTracker）JobTracker负责调度DataNode上的工作。每个DataNode
有一个TaskTracker，它们执行实际工作。
5）NodeManager（TaskTracker）执行任务。
6）DFSZKFailoverController高可用时它负责监控NN的状态，并及时的把状态信息写入ZK。
它通过一个独立线程周期性的调用NN上的一个特定接口来获取NN的健康状态。FC也有选择谁作
为Active NN的权利，因为最多只有两个节点，目前选择策略还比较简单（先到先得，轮换）。
7）JournalNode 高可用情况下存放namenode的editlog文件。
```

##### hadoop宕机

```tex
1）如果MR造成系统宕机。此时要控制Yarn同时运行的任务数，和每个任务申请的最大内存。调整参数：yarn.scheduler.maximum-allocation-mb（单个任务可申请的最多物理内存量，默认是8192MB）
2）如果写入文件过快造成NameNode宕机。那么调高Kafka的存储大小，控制从Kafka到HDFS的写入速度。例如，可以调整Flume每批次拉取数据量的大小参数batchsize。
```

##### hadoop1和2 区别

```tex
  1）加入了yarn解决了资源调度的问题，实现了运行的用户程序与yarn框架完全解耦。
  2）加入了对zookeeper的支持实现比较可靠的高可用。
```

##### 在HDFS写入数据，写某一副本出错时HDFS的处理流程

```tex
1.首先会关闭管线pipeline。
2.随后将已经发送到管道中但是没有收到确认的数据包重新写回数据队列，这样无论哪个节点发生故障，都不会发生数据丢失。
3.然后当前正常工作的数据节点将会被赋予一个新的版本号（利用namenode中租约的信息可以获得最新的时间戳版本），这样故障节点恢复后由于版本信息不对，故障DataNode恢复后会被删除。
4.在当前正常的datanode中根据租约信息选择一个主DataNode，并与其他正常DataNode通信，获取每个 DataNode当前数据块的大小，从中选择一个最小值，将每个正常的DataNode同步到该大小。然后重新建立管道。
5.在管线中删除故障节点，并把数据写入管线中剩下的正常的DataNode，即新的管道。
6.当文件关闭后，namenode发现副本数量不足时会在另一个节点上创建一个新的副本。
```

##### hdfs小文件的影响和处理

```tex
元数据层面：每个小文件都有一份元数据，其中包括文件路径，文件名，所有者，所属组，权限，创建时间等，这些信息都保存在Namenode内存中。所以小文件过多，会占用Namenode服务器大量内存，影响Namenode性能和使用寿命
计算层面：默认情况下MR会对每个小文件启用一个Map任务计算，非常影响计算性能。同时也影响磁盘寻址时间。
处理前可以：使用har工具将小文件进行归档，
  计算中， 2. 采用 CombineTextInputFormat 将多个小文件的从逻辑上规划到一个切片中，有一个maptask来进行处理
    驱动类中设置：
     job.setInputFormatClass(CombineTextInputFormat.class)
     CombineTextInputFormat.setMaxInputSplitSize(job,4194304)
   计算前： 3.设置jvm重用 mapreduce.job.jvm.numtasks=15 避免jvm频繁启停

```

##### secondary namenode工作机制

```tex
第一阶段：NameNode启动
  （1）第一次启动NameNode格式化后，创建fsimage和edits文件。如果不是第一次启动，直接加载编辑日志和镜像文件到内存。
  （2）客户端对元数据进行增删改的请求。
  （3）NameNode记录操作日志，更新滚动日志。
  （4）NameNode在内存中对数据进行增删改查。
2）第二阶段：Secondary NameNode工作
  （1）Secondary NameNode询问NameNode是否需要checkpoint。直接带回NameNode是否检查结果。
  （2）Secondary NameNode请求执行checkpoint。
  （3）NameNode滚动正在写的edits日志。
  （4）将滚动前的编辑日志和镜像文件拷贝到Secondary NameNode。
  （5）Secondary NameNode加载编辑日志和镜像文件到内存，并合并。
  （6）生成新的镜像文件fsimage.chkpoint。
  （7）拷贝fsimage.chkpoint到NameNode。
  （8）NameNode将fsimage.chkpoint重新命名成fsimage。
```

##### hadoop作业提交流程

```tex
1.客户端提交一个mr的jar包给JobClient(提交方式：hadoop jar ...)
2. JobClient向ResourceManager发送一个RPC请求，ResourceManager返回一个JobID和一个存放jar包的路径给Client
3. Client将得到的jar包的路径作为前缀，JobID作为后缀(path = hdfs上的地址 + jobId) 拼接成一个新的hdfs的路径，然后Client通过FileSystem向hdfs中存放jar包，默认存放10份（NameNode和DateNode等操作）
4. 开始提交任务，Client将作业的描述信息（JobID和拼接后的存放jar包的路径等）RPC返回给ResourceManager
5. ResourceManager进行初始化任务，然后放到一个调度器中
6. ResourceManager读取HDFS上的要处理的文件，开始计算输入分片，每一个分片对应一个MapperTask，根据数据量确定起多少个mapper,多少个reducer
7. NodeManager 通过心跳机制向ResourceManager领取任务（任务的描述信息）
8. 领取到任务的NodeManager去Hdfs上下载jar包，配置文件等
9. NodeManager启动相应的子进程yarnchild，运行mapreduce，运行maptask或者reducetask
10. map从hdfs中读取数据，然后传给reduce，reduce将输出的数据给回hdfs
```



