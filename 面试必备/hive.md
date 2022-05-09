[TOC]

##### hive的架构

```tex
由用户连接接口，元数据管理，驱动器组成
1.用户连接接口： 1.Shell命令行 2.JDBC/ODBC:是指Hive的java实现，与传统数据库JDBC类似，WebUI:是指可通过浏览器访问Hive。
2.元数据 ：Hive将元数据存储在数据库中，如mysql、derby。Hive中的元数据包括(表名、表所属的数据库名、 表的拥有者、列/分区字段、表的类型（是否是外部表）、表的数据所在目录等）
3.驱动器(Driver)包括解析器，编译器，优化器以及执行器；
    解析器(SQLParser): 将HQL字符串转换成抽象语法树AST，这一步一般都用第三方工具库完成，比如antlr；对AST进行 语法 分析，比如表是否存在、字段是否存在、SQL语义是否有误。
    编译器(Compiler)： 对hql语句进行词法、语法、语义的编译(需要跟元数据关联)，编译完成后会生成一个执行计划。 hive上就是编译成mapreduce的job。
    优化器(Optimizer)： 将执行计划进行优化，减少不必要的列、使用分区、使用索引等。优化job。
    执行器(Executer): 将优化后的执行计划提交给hadoop的yarn上执行。提交job。
```

##### hive的工作原理

```tex
 提交HQL到hive-client, client连接驱动程序driver,驱动程序中的解释器对HQL进行解析形成抽象语法树；编译器通过matestors连接元数据库对HQL的数据类型及语法分析，形成逻辑计划；优化器进行优化生成MR任务；执行器执行MR任务。
```



##### hive常用函数

```tex
字符串类型的
	length(string A)    int ：字符串长度函数
	concat（string A, string B…） ：字符串连接函数
	concat_ws(string SEP, string A, string B…） ： 按指定分隔符进行分割
	substring(string A, int start, int len)  ： 字符串截取
	 regexp（润转可s）_replace（string A, string B, string C） ：正则表达式替换函数
	 get_json_object（string json_string, \$string path）：
	 split(string str, string pat) array :分割字符串函数
聚合函数：sum(),avg(),max(),min(),count()
数值计算类型的
	round(double a)  BIGINT ：返回指定精度d的double类型 （遵循四舍五入）
	floor(double a)         ： 向下取整函数: floor
	ceil(double a)          :  向上取整函数
	rand(int seed)			:取随机数函数
	abs(double a)			:绝对值函数
日期类型的：
	from_unixtime(bigint unixtime[, string format]) string：时间戳转日期函数
	unix_timestamp(string date,string pattern) bigint ：日期转UNIX时间戳
	to_date(string timestamp) string ：返回日期时间字段中的日期部分。
	year（string date） month （string date） day（string date）
m	datediff(string enddate, string startdate) ：日期比较函数
	date_add(string startdate, int days)：
	 date_sub (string startdate, int days) ：日期减少函数
	 add_month(string date, int month):指定月份增加函数
开窗函数
（1）RANK()排序相同时会重复，总数不会变 
（2）DENSE_RANK()排序相同时会重复，总数会减少 
（3）ROW_NUMBER()会根据顺序计算 
2） OVER()：指定分析函数工作的数据窗口大小，这个数据窗口大小可能会随着行的变而 
变化
（5） LAG(col,n)：往前第 n行数据 
（6）LEAD(col,n)：往后第 n行数据 
（7） NTILE(n)：把有序分区中的行分发到指定数据的组中，各个组有编号，编号从 
1开始，对于每一行，NTILE返回此行所属的组的编号。注意：n必须为 int类型。 
	
```

##### hive优化

```tex
Hive优化
4.个方面 表关联方面，sql语句方面的话，3.mr方面的话 4.其他参数方面
1.表关联方面：
 1）大表和小表进行join的话使用mapjoin
    原理: 先启动一个local taskA负责扫描小表的数据将其转换成一个hashTable的数据结构，并写入本地的文件当中，之后将该文件加载到DistributeCache中，另一个maptask(没有reduce的mr)扫描大表，在map阶段，根据a的每一条记录去和distributeCache中b表对应的HashTable关联，并直接输出结果，有多少个maptask就有多少个结果文件。
  2）大表与大表进行jion 可以进行空key的转换，消除数据倾斜，或者提前进行过滤，减少join的数据量
  3）表关联之前进行条件的过滤
  4）使用分区表，设置动态分区 
   开启动态分区功能默认true
	hive.exec.dynamic.partition=true
  设置为非严格模式(动态分区的模式，默认strict，表示必须指定至少一个分区为静态分区，nonstrict模式表示允许所以的分区字段都可以使用动态分区)
	hive.exec.dynamic.partition.mode=nonstrict
	在所以执行MR的节点上，最大一共可以创建多少个动态分区
	Hive.exec.max.dynamic.partitions=100
	在每个执行MR的节点上，最大可以创建多少动态分区。
	hive.exec.max.dynamic.partitions.pernode=100
  5）使用分桶表，进行抽样，避免数据倾斜
    建表使用关键字 clustered by(分桶字段)
     into 4 buckets
      开启分桶
    set hive.enforce.bucketing=true
    查询用到的函数  
    抽样:tablesample(bucket x out of y on id)
    
 2.sql语句方面的话：
 	1.查询只读自己查询时只读取需要的列，需要的分区
 	1.sort by代替order by  order by是全局排序，性能较低，sort by 只是分区内排序
 	2.尽量避免使用Count(distinct)去重，因为只会启动一个reduce，处理的数据量过大
	可以group by 在count
	3.开启map端聚合，减少数据的传输量，hive.map.aggr=true 
	  开启有数据倾斜的时候进行负载均衡；Hive.groupby.skewindata=true
	  
 3.mr方面的话
	3) 控制设置mapreduce的个数
	 1. map个数控制，
	set maprduce.input.fileinputformat.split.maxsize=100m   最大切片值设置 默认256m
注意如果使用的默认压缩算法，不支持文件切分，但支持文件合并，设置split小于文件大小是不生效的
可以考虑使用lzo压缩，但是需要建立索引文件,
    2. reduce个数设置 min(每个任务最大的reduce数999,主要根据总的数据量/reduce处理的数据量）
    手动调整
    set mapred.reduce.tasks＝15
	3）小文件进行合并，可以在map执行之前合并小文件，减少map数：
		1.采用 har归档方式，将小文件归档（处理的是文件已经上传到了 HDFS上了）
		2.开启Jvm重用， mapreduce.job.jvm.numtasks=15 避免jvm频繁启停
        Set mapred.job.reuse.jvm.num.tasks =20或在Hadoop的mapred-site.xml配置
        3.默认采用的是hive
	4）也可以通过抽样将倾斜的key取出来，单独处理，然后在union回去
	5）启用压缩： 开启Map和reduce输出阶段压缩
     6）使用orc、或parquet文件格式
     	Orcfile:列式存储，有多重文件压缩格式，并且有很高的压缩比，支持切分的，提供了多种索引，支持复杂的数据结构
		parquet：以二进制方式存储的，列式存储，存储嵌套数据，兼容hadoop大多数的计算框架（MR,SPARK等），支持多种查询引擎（hive，Impala）
	在hive中 orc默认的压缩格式是ZLIB压缩
	7）使用Tez引擎，是一个基于DAG任务的数据处理框架，把mapreduce的过程拆分成若干个子过程，把多个mapreduce任务组合成一个较大的DAG任务，这样一方面减少了mapreduce之间的文件存储，另一方面合理组合其子过程从而大幅提升MapReduce作业的性能。
	TEZ和MR的区别：
	1.TEZ是基于内存的，所有对内存要求比较高，MR的话，是基于磁盘，可以确保任务的完成
	2.资源分配方面的话：TEZ会拿到预估的资源，在结束计算后释放，可以通过参数调整；
    	MR会通过解析任务步骤，释放、申请、释放这样的执行。
	
 4.其他参数方面
  1.设置并行执行的线程数 set hive.exec.parallel=true
    同一个sql允许最大并行度
    Set hive.exec.parallel.thread.number=16   8
  2. 开启严格模式 set hive.mapred.mode=static  nonstrict
  限制了笛卡尔积的查询
  使用order by 语句必须跟上limit 语句
  分区表必须 where语句中含有分区字段过滤条件来限制范围
  3. 开启本地模式
    set hive.exec.mode.local.auto=true
    开启本地mr
    set hive.exec.mode.local.auto.inputbytes.max=50000000
    设置localmr的最大输入数据量，当数据量小于这个值采用local mr的方式，默认128m
    set hive.exec.mode.auto.input.files.max=10 默认4 
  4. 推测执行
  5. 执行计划

```

##### UDF、udaf，udtf以及    四个by的区别

```tex
1）UDF函数(一进一出)：日期字段，json字段。（手机app项目）
udaf（多进一出）：
udtf（一进多出）：
2）用UDF函数：对数据解析字段，对时间初始化的类，根据输入的类型，判断是否在这个时间范围内，类型可能是day，week，以及month
主要是使用了java的canlinder类和Date和SimpleDateFormat
自定义UDF：继承UDF，重写evaluate方法，打成jar包，上传到hdfs上，
在hive中使用add jar 路径 命令 添加到类启动路径中，
create function name as 主类的全路径包名
自定义UDTF：继承自GenericUDTF，重写3个方法：initialize(自定义输出的列名和类型)，process（将结果返回forward(result)），close


1）Sort By：分区内有序；
2）Order By：全局排序，只有一个Reducer；
3）Distrbute By：类似MR中Partition，进行分区，结合sort by使用。
4） Cluster By：当Distribute by和Sorts by字段相同时，可以使用Cluster by方式。Cluster by除了具有Distribute by的功能外还兼具Sort by的功能。但是排序只能是升序排序，不能指定排序规则为ASC或者DESC。
```

#####  hive（数据仓库和数据库（软件mysqloracle)比较

```tex
Hive 和数据库除了拥有类似的查询语言，再无类似之处。
用处：数仓主要用来进行olap 联机分析处理，数据库主要进行oltp 联机事务分析处理
1）数据存储位置
Hive 存储在 HDFS 。数据库将数据保存在块设备或者本地文件系统中。
2）数据更新
Hive中不建议对数据的改写。而数据库中的数据通常是需要经常进行修改的， 
3）执行延迟
Hive 执行延迟较高。数据库的执行延迟较低。当然，这个是有条件的，即数据规模较小，当数据规模大到超过数据库的处理能力的时候，Hive的并行计算显然能体现出优势。
4）数据规模
Hive支持很大规模的数据计算；数据库可以支持的数据规模较小。
```

##### 文件格式orc、sequenceFile、parquet

```tex
文件格式
sequenceFile:是hadoopapi提供的一种二进制压缩文件,将数据以key value的形式序列化到文件中，他的底层使用Hadoop的标准的writable接口来实现
RCfile:hive推出的一种面向列的存储格式,在查询的过程中，针对它并不关心的列,它会在io上跳过这些列
Orcfile:列式存储，有多重文件压缩格式，并且有很高的压缩比，支持切分的，提供了多种索引，支持复杂的数据结构
parquet：以二进制方式存储的，列式存储，存储嵌套数据，兼容hadoop大多数的计算框架（MR,SPARK等），支持多种查询引擎（hive，Impala）
在hive中 orc默认的压缩格式是ZLIB压缩
```

##### hive内部表和外部表

```tex
内部表和外部表
创建表时：创建内部表时，会将数据移动到数据仓库指向的路径。创建外部表时 ，EXTERNAL关键字修饰，仅记录数据所在的路径，不对数据的位置做任何改变，
删除表时：在删除表的时候，内部表的元数据和数据会被一起删除，而外部表只删除元数据，不删除数据

我们采集的数据使用的就是外部表，一旦被误删，恢复起来非常麻烦
做统计分析用到的中间表，结果表都是内部表，因为这些数据不需要共享
修改内部表student2为外部表
alter table student2 set tblproperties('EXTERNAL'='TRUE');
```

##### hive解决数据倾斜？

```tex
原因：不同数据类型关联产生数据倾斜 ，key分布不均匀，业务数据本身的特性：未控制null值的分布，某些SQL语句本身就有数据倾斜，不可拆分大文件引发的数据倾斜；
处理：1.不同数据类型关联产生的可以在进行join之前进行一个类型转换 cast( as )
	2.sql语句倾斜，比如count(distinct),采用 sum()groupby的方式来替换 count(distinct)完成计算, 或者使用mapjoin，避免在reduce进行聚合，产生数据倾斜，
	3.开启数据倾斜时负载均衡 ：Hive.groupby.skewindata=true
	生成的查询计划会有两个 MRJob。 
第一个 MRJob中，Map的输出结果集合会随机分布到 Reduce中，每个 Reduce做部 
分聚合操作，并输出结果，这样处理的结果是相同的 GroupByKey有可能被分发到不同的 
Reduce中，从而达到负载均衡的目的； 
第二个 MRJob再根据预处理的数据结果按照 GroupByKey分布到 Reduce中（这个过 
程可以保证相同的原始 GroupByKey被分布到同一个 Reduce中），最后完成最终的聚合操 
作。
	4.不可拆分大文件引发的数据倾斜
	我们在对文件进行压缩时，在可以采用bzip2和lzo等支持文件分割的压缩算法。
	5.表连接时引发的数据倾斜，使用mapjoin，在map端进行局部聚合，避免reduce端join
	6.未控制null值的分布，将为空的 key转变为字符串加随机数或纯随机数，将因空值而造成倾斜的数据分不到多个 Reducer。 对于异常值，如果不需要的话，提前在where条件中过滤。
	7.业务本身的特性的话，进行抽样查询，将倾斜的key单独处理，然后在union到业务数据表中
```



