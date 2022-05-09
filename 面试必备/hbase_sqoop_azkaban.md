##### hbase是什么

```tex
Apache HBase是一个开源的、分布式的、版本化的、非关系型数据库，他的特点主要有：

1）海量存储：适合存储PB级别的海量数据，在PB级别的数据以及采用廉价PC存储的情况下，能在几十到百毫秒内返回数据
2）极易扩展：通过横向添加RegionSever的机器，进行水平扩展，提升Hbase上层的处理能力，提升Hbsae服务更多Region的能力。
3）面向列：hbase是根据列族来存储数据的，列族下面可以有很多的列
4）稀疏：列族中，我们可以指定任意多的列，在列数据为空的情况下，是不会占用存储空间的。
5）数据多版本：每个单元中的数据可以有多个版本，默认情况下版本号自动分配，是单元格插入时的 时间戳
6）数据类型单一：Hbase中的数据都是字符串，没有类型
```

##### hbase和hive的区别

```tex
	hbase，以k-v形式存储的列式数据库，使用的数据库引擎，支持增删改查，需要预先定义列族，不需要具体到列，主要应用于实时的场景	
	数据仓库面向主体，集成的，反应历史变化的数据集合，使用的是mr引擎，只支持导入和查询，需要预先定义表格，适用于离线处理
```

##### HBase的rowKey的设计原则

```tex
1）Rowkey长度原则
rowkey是一个二进制码流，可以为任意字符串，最大长度为64kb,一般控制在10-100bytes
原因如下：
（1）数据的持久化文件HFile 中是按照KeyValue 存储的，如果Rowkey 过长比如100 个字节，1000 万列数据光Rowkey 就要占用100*1000 万=10 亿个字节，将近1G 数据，这会极大影响HFile 的存储效率
（2）MemStore 将缓存部分数据到内存，如果Rowkey 字段过长内存的有效利用率会降低，系统将无法缓存更多的数据，这会降低检索效率。
（3）目前操作系统是都是64 位系统，内存8 字节对齐。控制在16 个字节，8 字节的整数倍利用操作系统的最佳特性
2）Rowkey散列原则，rowkey均匀的分布在hbase节点上，
	hbase
举个例子，如果Rowkey 是按时间戳的方式递增，不要将时间放在二进制码的前面，建议将Rowkey的高位作为散列字段，由程序循环生成，低位放时间字段，这样将提高数据均衡分布在每个Regionserver   实现负载均衡的几率。如果没有散列字段，首字段直接是时间信息将产生所有新数据都在一个 RegionServer 上堆积的热点现象，这样在做数据检索的时候负载将会集中在个别RegionServer，降低查询效率
热点问题处理：1.对于固定长度的RowKey反转后存储，比如手机号，开头比较固定，就可以进行反转，避免热点问题，但是牺牲了rowkey的有序性
	2.加盐：给每一个rowkey加一个随机前缀，时数据分散在不同region，增加了写操作的吞吐量，缺点增加了读操作的开销
	3.hash处理：根据rowkey计算其hash值, 在rowkey前面hash计算值即可，让相关性比较强的数据可以被放置到同一个region中，缺点，相关数据较多，还有可能产生热点问题
3）Rowkey唯一原则,必须在设计上保证其唯一性
因为HBase中数据存储是Key-Value形式，若HBase中同一表插入相同Rowkey，则原先的数据会被覆盖掉(如果表的version设置为1的话)，所以务必保证Rowkey的唯一性.

```

##### hbase的架构

```tex
Client
Client包含了访问Hbase的接口，另外Client还维护了对应的cache来加速Hbase的访问，比如cache的.META.元数据的信息。
2）Zookeeper
HBase通过Zookeeper来做master的高可用、RegionServer的监控、元数据的入口以及集群配置的维护等工作。
具体工作如下：
通过Zoopkeeper来保证集群中只有1个master在运行，如果master异常，会通过竞争机制产生新的master提供服务
通过Zoopkeeper来监控RegionServer的状态，当RegionSevrer有异常的时候，通过回调的形式通知Master RegionServer上下线的信息
通过Zoopkeeper存储元数据的统一入口地址
3）Hmaster：
    1. 管理用户对Table的增、删、改、查操作
    2. 管理HRegionServer的负载均衡，调整Region分布
    3. 在Region Split后，负责新Region的分配
    4. 在HRegionServer停机后，负责失效HRegionServer 上的Regions迁移
4）HregionServer
    HregionServer直接对接用户的读写请求，是真正的“干活”的节点。
    管理master为其分配的Region
    处理来自客户端的读写请求
    负责和底层HDFS的交互，存储数据到HDFS
    负责Region变大以后的拆分
    负责Storefile的合并工作
5）HDFS
HDFS为Hbase提供最终的底层数据存储服务，同时为HBase提供高可用（Hlog存储在HDFS）的支持，具体功能概括如下：
提供元数据和表数据的底层分布式存储服务
数据多副本，保证的高可靠和高可用性
6)其他组件：
	1．Write-Ahead logs
HBase的修改记录，当对HBase读写数据的时候，数据不是直接写进磁盘，它会在内存中保留一段时间（时间以及数据量阈值可以设定）。为了避免数据丢失，数据会先写在一个叫做Write-Ahead logfile的文件中，然后再写入内存中。所以在系统出现故障的时候，数据可以通过这个日志文件重建。
	2．Region
Hbase表的分片，HBase表会根据RowKey值被切分成不同的region存储在RegionServer中，在一个RegionServer中可以有多个不同的region。
	3．Store
HFile存储在Store中，一个Store对应HBase表中的一个列族。
	4．MemStore
顾名思义，就是内存存储，位于内存中，用来保存当前的数据操作，所以当数据保存在WAL中之后，RegsionServer会在内存中存储键值对。
	5．HFile
这是在磁盘上保存原始数据的实际的物理文件，是实际的存储文件。StoreFile是以Hfile的形式存储在HDFS的。orc 的文件格式。
```

##### HBase中compact用途是什么，什么时候触发，分为哪两种，有什么区别，有哪些相关配置参数？

```tex
在hbase中每当有memstore数据ﬂush到磁盘之后，就形成一个storeﬁle，当storeFile的数量达到一定程度后，就需要将 storeﬁle 文件来进行 compaction 操作
Compact 的作用：
1）合并文件
2）清除过期，多余版本的数据
3）提高读写数据的效率
HBase 中实现了两种 compaction 的方式：minor(卖哪) and major（美聚）. 这两种 compaction 方式的区别是：
1）Minor 操作只用来做部分文件的合并操作以及包括 minVersion=0 并且设置 ttl 的过期版本清理， 不做任何删除数据、多版本数据的清理工作
2）Major 操作是对 Region 下的HStore下的所有StoreFile执行合并操作，最终的结果是整理合并出一个文件
```

##### HBase读写流程

```tex
读：
1.HRegionServer保存着meta表以及表数据，要访问表数据，首先Client先去访问zookeeper，从zookeeper里面获取meta表所在的位置信息，即找到这个meta表在哪个HRegionServer上保存着。
2.接着Client通过刚才获取到的HRegionServer的IP来访问Meta表所在的HRegionServer，从而读取到Meta，进而获取到Meta表中存放的元数据。
3.Client通过元数据中存储的信息，访问对应的HRegionServer，然后扫描所在HRegionServer的Memstore和Storefile来查询数据。
4. 最后HRegionServer把查询到的数据响应给Client。
写：===========
1. Client先访问zookeeper，找到Meta表，并获取Meta表元数据。
2.确定当前将要写入的数据所对应的HRegion和HRegionServer服务器。
3.Client向该HRegionServer服务器发起写入数据请求，然后HRegionServer收到请求并响应。
4.Client先把数据写入到HLog，以防止数据丢失。
5.然后将数据写入到Memstore。
6.如果HLog和Memstore均写入成功，则这条数据写入成功
7.如果Memstore达到阈值，会把Memstore中的数据flush到Storefile中。	
8.当Storefile越来越多，会触发Compact合并操作，把过多的Storefile合并成一个大的Storefile。
9.当Storefile越来越大，Region也会越来越大，达到阈值后，会触发Split操作，将Region一分为二。
```

