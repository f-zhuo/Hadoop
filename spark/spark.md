spark是一个分布式的并行计算框架，是下一代的MapReduce，比起MapReduce的优点在于：

* mr是多进程，调度慢，需要启动map和reduce两个任务；spark是多线程，响应快

* mr计算慢，每一步都要保存中间结果，存入磁盘；spark多是在内存中流转数据，计算更快

* mr的API简单，只有map和reduce，缺乏作业流，需多轮mr；spark算子丰富，作业可以由语句串起来

*多进程：进程是资源（CPU，内存等）分配的基本单元，每个进程都有独立的地址空间，不会资源抢占，一个进程失败了也不会影响其他进程。但是启动时间更长，有延时性*

*多线程：线程是执行程序的基本单元，是CPU分配资源的基本单元。启动快，同一进程的所有线程的内存地址相同，所以共享内存，适合内存密集型任务（多词表），但是会出现资源抢占的情况，一个线程失败了会影响其他线程*

启动：

```bash
./sbin/start-all.sh
```

# spark的部署方式

## 本地模式

不使用集群

```bash
./bin/spark-submit \
--class org.apache.spark.examples.SparkPi \ # 作业代码包
--master local[2] \ # 2表示并发数
lib/spark-examples-1.6.3-hadoop2.6.0.jar # 依赖的jar包
```

## Spark standalone

使用spark本身的集群，可以不依赖Hadoop2.x和yarn集群运行。master/worker架构，不存在单点故障，由zookeeper解决（类似于hbase的单点故障）。运行任务的地方和MR1一样，都是slot，但是不区分类型

```bash
./bin/spark-submit  --class org.apache.spark.examples.SparkPi --master spark://master:7077  lib/spark-examples-1.6.3-hadoop2.6.0.jar  # 10表示task个数
```

## Spark on yarn

依赖yarn集群，与Hadoop共享平台

```bash
./bin/spark-submit --class org.apache.spark.examples.SparkPi --master yarn-cluster lib/sparkexamples-1.6.3-hadoop2.6.0.jar 1000 # 1000表示task个数

./bin/spark-submit --class org.apache.spark.examples.SparkPi --master yarn-client lib/sparkexamples-1.6.3-hadoop2.6.0.jar 10
```

Spark on yarn 有两种模式：

* yarn-cluster：适合生产环境，AM(driver)在某个**NM**启动并提交作业，向RM申请的资源是executor，用户提交了作业后会关闭client，作业在yarn上进行，只能通过log查看

* yarn-client：适合交互和调试，AM(driver)在**本地**启动并提交作业的，用户提交作业后，依靠client和container进行通信作业，不会关闭client

## spark on mesos

也分为两种：

* 粗粒度模式 （Coarse-grained Mode）：application的每个任务在运行之前就分配好了全部资源，运行期间会一直占用，哪怕有资源不被使用，等到application完成之后才会释放
* 细粒度模式 （Coarse-grained Mode）：一开始executor只会占用自身启动的资源，采用动态分配的方法，需要多少分配多少，只要自身的任务完成就会释放

*不管是哪种模式，数据都是存储在hdfs，输入/输出文件路径也是*

# spark架构

## Spark on yarn

* RM：resource manager，布署在yarn上，负责分配和调度资源

* NM：node manager，节点的资源和任务管理器，启动executor和application master

* AM：application master，又叫driver，每个application都有一个，负责的是申请资源，分配和监视任务

## Spark standalone

* master：类似RM

* worker：类似NM

# spark作业

Spark和MapReduce作业的区别：

MapReduce的每一个程序就是一个job，一个job包括多个task，有map task和reduce task。因为是多进程，每个task在自己的进程运行，一旦task完成进程就结束

![](.\pictures\job.png)

spark中，提交的程序是Application，可以包括多个job。一个action算子就是一个 job，一个job会切换成一定数量的stage。各个stage之间按照顺序执行，是上下游的关系。task是spark中最小的工作单元，可并发执行

![](.\pictures\application.png)

spark的application由一个driver和多个job构成

* driver：其实就是application master，是一个驱动程序，是Spark的核心组件

  * 构建SparkContext：

    sc，spark应用入口，编写spark程序用到的第一个类，包含sparkconf，sparkenv等类，负责与spark集群进行交互（申请资源、创建RDD、广播。。。），创建需要的变量，包含集群的配置信息等

  * 将用户提交的job转换为DAG图（类似数据处理的流程图），根据策略将DAG图划分为多个stage。一个stage对应一个taskset，taskset包含一组关联的没有shuffle依赖关系的task（task在stage里表现为partition），stage是通过shuffle/宽依赖区分的。若一个stage包含的其他stage中的任务已经全部完成，这个stage中的任务才会被加入调度

  * 根据task要求向RM申请资源 ，提交任务并监测任务状态。分配任务时遵循数据局部性原则，要求数据传输代价最小，如果一个任务需要的数据在某个节点的内存中，这个任务就会被分配至那个节点

  ![](.\pictures\job-stage.png)



* Executor：真正执行task的单元，作为一个YARN容器 (container)运行。executor能分配到多少task由AM决定的，可以看作是运行多个task的线程池。一个节点上可以有多个Executor，每个Spark executor container默认内存是1G，由参数yarn.scheduler.minimum-allocation-mb定义。executor分配内存的参数是executor-memory，向YARN申请的内存是executor-memory * num-executors 

## 执行过程

driver启动一个SC，向RM申请资源，集群主节点会启动几个从节点，在从节点上启动excutor进程，分配给资源并执行task任务，从节点会直接连接到driver上进行交互。每一个action算子就是一个job，driver把job转换为DAG，再划分为stage，一个stage对应一个task set，将task set的每一个task交给executor执行，task完成之后形成新的RDD，再传给driver循环操作直至所有算子完成

spark启动涉及的参数：

* executor-memory：每个executor内存多大，如果执行内存偏多，通过设置调大一些，比如80%

* num-executors：多少个executor进程

* executor-cores：每个exector进程虚拟的core cpu个数，最好不要设为1，一般设置2~4。executor-cores * num-executors = 一共需要多少个虚拟core，一般不要超过总队列的25%

提交任务：

```scala
./bin/spark-submit \
 --class org.apache.spark.examples.SparkPi
 --master yarn-cluster \
 --num-executors 100 \
 --executor-memory 6G \
 --executor-cores 4 \
 ...
 lib/sparkexamples-1.6.3-hadoop2.6.0.jar 1000
```

杀死任务

```bash
yarn application -kill application_xxxxxx_yyy
```

# 内存管理

* 有存储级别：判断是否有缓存，先缓存后磁盘

* 无存储级别：直接磁盘读

executor内存分为3部分：

* execution内存（60%）：执行内存，执行shuffle的时候，shuffle会用这个内存区来存储数据，如果溢出，写磁盘。如果两个表连接，一个表大，一个表小，可以把小的表放在内存中，用来作字典，查找更快

* storage内存（20%）：存储缓存，cache、presist、broadcast，防止子RDD失效

* other内存（20%）：应用程序的内存，同时作为execution和storage的误差缓冲

spark1.6以前，内存之间互相隔离，导致利用率不高；1.6版本之后，execution和storage之间可以相互借用，减少OOM（溢出/out of memory）情况

# spark算子

## transformation

转换算子：转换并不是触发提交，只是完成作业中间过程处理，转换操作不是马上执行，需要等到有 Acion 操作的时候才会真正触发运算，称为延迟计算/懒惰机制

分类：

* 输入输出分区的对应关系：
  * 一对一：map，flatmap
  * 多对一：union，cartesian
  * 多对多：groupby

* 输出是否是输入子集合：filter，distinct，sample

* cache类：cache，persist

* 聚集：reduceByKey，combineByKey，partitionBy

* 连接：join，leftOutJoin，rightOutJoin

## action

行动算子：触发提交作业，可以将结果输出到hdfs，hbase，kafka或者console终端等

* 无输出：foreach

* 有输出：saveAsTextFile

* 统计类：count，collect，take

 ![](.\pictures\算子.png)

![](.\pictures\算子2.png)

# RDD

弹性分布式数据集（ Resilient Distributed Dataset ），只读且可分区。本质是数据集的描述，而不是数据集本身，只存储数据的分区信息（partition）和读取方法，不是数据，也不存数据

特点：

* 一般是partition的集合，每个partition都有函数，可以实现partition的转换。spark的partitioner有两种：hashpartitioner和rangepartitioner，前者是使用hash进行partition，是大部分transformation算子的实现方式，后者是根据数据的分布尽量均匀分区

* RDD可以计算变换得到另一个RDD，结果保存在内存中。但RDD是只读的，不能对本身修改

* 具有懒惰性，在action算子时才对数据进行一定的操作，transformation只是RDD间的变换

* 不包括待处理的数据，真正的数据只有在执行的时候才加载进来，用完就释放

* 用lineage记录数据的变换而不是数据本身，容错性好，一旦数据丢失，可以有足够的信息重新计算得到数据，但对于重要数据，还是会备份

lineage表现为DAG图

![](.\pictures\lineage.png)

## 构建RDD的途径

* 从共享文件系统获取，如HDFS读取

```scala
val rdd1 = sc.textFile(“/xxx/file”)
```

* 通过其他RDD转换

```scala
val rdd2 = rdd1.map(x => (x, 1))
```

* 通过定义Scala数组

```scala
val rdd3 = sc.parallelize(1 to 10, 1)
```

* 通过其他RDD的持久化操作

```scala
val rdd4 = rdd1.persist()
rdd1.saveAsHadoopFile(“/xxx”)
```

## RDD间的依赖

顶部的RDD称为数据源，非顶部RDD用lineage（血统）记录自己来源于谁，子RDD向上依赖于父RDD

* 宽依赖：子RDD的partition依赖于父RDD的所有partition，比如groupbykey，reducebykey，join，由父RDD产生子RDD时会先对父RDD进行shuffle分桶。一旦失效，需要对整个RDD的所有丢失的父RDD进行计算；但若进行了缓存，就不需要

* 窄依赖：子RDD的partition依赖于父RDD的有限个partition，比如map，filter，union。失效时，只需要计算丢失RDD的父分区，不同节点可以并行

 ![](.\pictures\narrow_wide.png)

每个stage尽可能多得包含窄依赖关系，将宽依赖删掉，分离出来的就是一个stage

![](.\pictures\stage.png)

# spark容错

lineage记录RDD的依赖关系，对父RDD到子RDD的操作过程进行缓存或者重新计算父RDD

## spark standalone

### master

集群会启动多个master，但是只有一个是active状态，其余都是standby，当active失效时，要启用新的master。有以下几种方法：

* zookeeper：zk启用选举机制，选出新的master。但同时，为了保证高可用，需要提前将集群的元数据持久化到zookeeper中，zookeeper将元数据给新的master
* filesystem：集群的元数据是持久化到本地的文件系统的，只需要重启节点，master就能从本地获取元数据，恢复集群
* custom：用户自定义方法恢复集群
* none：不持久化集群元数据，而是直接由新的master接管集群

### worker

worker以心跳的方法告知master自身的状态，若失效，还要判断是driver还是executor。若是driver，需要重启，否则就直接删除

### executor

task如果失败，AM会重新分配task

# spark开发调优

* 避免创建重复的RDD：对同一份数据，应该只创建一个RDD，不能创建多个

```scala
// 错误写法
val rdd1 = sc.textFile("hdfs://192.168.0.1:9000/xxx.txt")
rdd1.map(...)
val rdd2 = sc.textFile("hdfs://192.168.0.1:9000/xxx.txt")
rdd2.reduceByKey(...)

// 正确写法
val rdd1 = sc.textFile("hdfs://192.168.0.1:9000/xxx.txt")
rdd1.map(...)
rdd1.reduceByKey(...)
```

* 尽可能复用同一个RDD： 比如，一个RDD数据格式是key-value，另一个是单独value类型，这两个RDD的value部分完全一样，可以复用减少算子执行次数

```scala
// 错误写法
val rdd1 = ...
rdd2 = rdd1.map(...)

//正确写法
val rdd1 = ...
rdd1.reduceByKey(...)
rdd1.map(tuple._2...)
```

* 对多次使用的RDD进行持久化处理：每次对一个RDD执行一个算子操作时，都会重新从源头处理计算一遍，直到计算出那个RDD，然后进一步操作，这种方式性能很差。可以对多次使用的RDD进行持久化，将RDD的数据保存在内存或磁盘中，避免重复计算，借助cache()和persist()方法

```scala
val rdd1 = sc.textFile(...).cache()
rdd1.map(...)

val rdd1 = sc.textFile(...).persist(StorageLevel.MEMORY_AND_DISK_SER)
rdd1.map(...)
```

cache和persist之间的关系：

cache只放内存；persist有很多参数选择，包括内存和磁盘，所以cache属于persist一种特殊形式

persist持久化的级别：

![](.\pictures\持久化级别.png)

* 避免使用shuffle类算子：在spark作业运行过程中，最消耗性能的地方就是shuffle过程。将分布在集群中多个节点上的同一个key，拉取到同一个节点上，进行聚合和连接处理，比如groupByKey、reduceByKey、join等算子，都会触发shuffle

  join改成Broadcast+map，不会导致shuffle，但适合RDD数据量较少时使用

```scala
//原写法
val rdd3 = rdd1.join(rdd2)

//写成
val rdd2Data = rdd2.collect()
val rdd2DataBroadcast = sc.broadcast(rdd2Data)
val rdd3 = rdd1.map(rdd2DataBroadcast...)
```

* 使用map-side预聚合的shuffle操作：

  * 只能使用shuffle无法用map类算子替代的，尽量使用map-site预聚合的算子
  * 思想类似MapReduce中的Combiner

  * 使用reduceByKey或aggregateByKey算子替代groupByKey算子，因为reduceByKey或aggregateByKey算子会使用用户自定义的函数对每个节点本地相同的key进行聚合

* 使用Kryo优化序列化性能：

  Kryo是一个序列化类库，来优化序列化和反序列化性能。Spark默认使用java序列化机制（ObjectOutputStream/ ObjectInputStream API）进行序列化和反序列化，Kryo序列化库性能比java序列化库高10倍左右

# pyspark

python的一个模块，可以直接使用spark的算子，不需要Scala 

进入pyspark

```bash
pyspark
```

执行pyspark作业

```bash
pyspark .py文件
```

pyspark分发tgz文件

* 命令行

```bash
pyspark x.py --py-files y.tgz
```

* 代码配置

```python
sc.addFile("y.tgz")
```
