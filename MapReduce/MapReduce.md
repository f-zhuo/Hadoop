MapReduce是一个分布式的计算框架，数据是存储在hdfs上的

# Hadoop出现之前处理大数据的方法

* 传统hash

把数据，流量均分到N台服务器

`Key(hash) mod N =0 `     分到第1台

`Key(hash) mod N =i`     分到第i+1台

`Key(hash) mod N =N-1` 分到第N台

当对应服务器挂掉时，该任务失败结束。这样做可以保证负载均衡

* 随机划分

数据，流量是随机分配到服务器上的

* 一致性hash（环形）

按照各台服务器的性能分配数据，流量，使数据和流量尽可能按各台服务器的最大负载量比例进行分配。当应当执行任务的服务器挂掉时，其他服务器就可以承担该任务（按性能比例承担），保证任务不会失败

 ![](.\pictures\consistent_hashing.png)

# 云计算的难点

* 单机系统变为分布式集群系统

* 保证稳定性和容错能力

* 保证数据一致性：节点上的数据更新，能保持一致

强一致：所有分布系统的节点都保持数据一致才会进行服务

弱一致：存在一个节点的数据更新完成后就可以进行服务，经过一段时间后才保证各节点数据一致，运行更快

本文介绍的是MR1，MR2的架构、作业提交和job的优先级都在yarn的介绍里

# MapReduce的计算原理

分而治之

比如：

小王经营一家奶茶店，假设只收现金，晚上打烊后计算一天的营业额。现金包括100元，50元，20元和10元纸币，由收银员**N**个人（任意人数）负责把自己今天的营业额按100，50，20，10元的纸币分类挑拣出，共**N**份，每份**四**堆；员工小明，小李，小王，小张**四**个人分别对N份的100元，50元，20元，10元的纸币堆计数，再乘以相应的金额并累加**N**份，分别得到按100元，50元，20元，10元计算的营业额，最后把四个营业额相加，得到了最终的营业额

*注：本例中可以再找小江一共5个人对分好的钞票堆计数，但是由于只有4种纸币，5个人有一个人是空闲的*

## MapReduce的计算流程

简单来说，就是：Map-shuffle-Reduce

其中，Map是分解过程，把输入文件的每一行作为一个输入值，键是行数（从0开始），值就是每一行的内容。但是，map函数只需要使用值，所以输出时键会被忽略掉，值按分隔符分开；Reduce是合并过程，接受map阶段输出的键值对，按键聚合计算；shuffle是一个哈希过程，把Map中的各个结果传递给不同的Reduce，在map输出和reduce输入的中间过程都可以认为是一个shuffle。map和reduce两个函数都以键值对作为输入和输出

![](.\pictures\MR.png)

![](.\pictures\MapReduce1.png)

### map

* split：

  从hdfs输出的数据经过数据切片（input split）变成若干等长的数据块block（压缩文件不可以被split），一个切片对应一个map

* map、partition：

  mapper接收block数据，使用map函数将其变为（key,value）的形式；按照key进行分区，把相同的key放在一个分区（partition）。事实上，每个数据被map时，都被HashPartition确定其partition。partition决定了它最终被哪个reducer计算，一个reducer可以计算多个partition。注意，这里的相同key的数据并不是全部在一个partition里。因为内存大小有限制，会发生多次溢写，每次溢写都有可能有相同的partition

  *partition的目的是将数据分给不同的reducer计算，达到负载均衡的效果*

* buffer in memory：

  将（key,value）连同partition写入环形内存缓冲区，同时，把block数据转换为record。内存默认大小是100M，当写入80%时，把内存中的数据输出并存储在磁盘里，这个过程叫溢写/spill

  *写入环形内存缓冲区的目的是避免大量的IO；使用环形内存缓冲区而非线性是因为可以同时进行读操作和写操作，溢写之后不需要移动缓冲区的数据就能继续写入；80%溢写而非100%是因为溢写需要时间，为了避免数据阻塞，要提前溢出*

* sort、combine：

  在溢写之前对每个partition进行sort和combine（combine默认不使用）。sort对同一partition的数据按key排序，combine下文详述

* spill、merge：

  当一个mapper的输出都溢写完毕之后，磁盘存在多个spill文件，会进行merge操作，每个mapper先将所有相同partition的文件进行merge为一个，然后将所有不同partition文件merge成一个，但partition依然没变。在对同一partition进行merge的过程中，也会进行sort

*block是hdfs存储数据的最小单元，Hadoop 1.x为64m,2.x为128m*

*map的输出默认不压缩，可以设置为压缩，减小数据量*

### reduce

copy、merge/sort、reduce：

因为每个partition的数据是由特定的reducer计算的，reduce task就会从所有mapper的输出中寻找特定的partition，将其copy过来（以HTTP的方式），进行merge，在merge的过程中又会按key去sort（每个partition是有序的，但partition之间未必有序），在sort后会使用reduce函数计算，当所有的partition都merge完毕，reduce计算完毕就得到自身的output。事实上，只要有一个map任务完成了，reduce就会开始任务。MapReduce的输出文件数等于reducer数，输出分为key和value两个部分，默认是以空格为分割符的，以分割的第一个字段为key，默认以key作为partition

下图为三个mapper输出的reduce过程

![](.\pictures\MapReduce2.png)

#### combine

将mapper的输出按partition给reducer的这一过程一般称为shuffle，该过程复杂，占用资源，为了优化，可以使用combine（归并）。所谓combine，就是在map过程中提前进行"reduce"

combiner会在每个mapper中进行reduce的操作，也就是按键聚合。最终reducer从每个mapper中得到的输入，相同键的键值对大大减少，简化了shuffle复杂度。但是应该慎用，有些情况不可用

*例*

数据集文件是小区业主的应缴电费，包括两列：第一列是门牌号，第二列是电费。假设每个记录是每个业主的每月应缴，电费三个月交一次，记录如下：

| 业主 | 电费 |
| :--: | :--: |
| 0001 |  34  |
| 0002 |  48  |
| 0001 |  27  |
| 0001 |  33  |
| 0003 |  53  |
| 0001 |  57  |
| 0001 |  39  |
| 0001 |  34  |

假设对于0001这个业主来说，六个数据分为两组在两个mapper里

**不采用combine计算电费和**

`mapper1：(0001,34)，(0001,27)，(0001,33) `

`mapper2：(0001,57)，(0001,39)，(0001,34) `

`reduce：(0001,[34,27,33,57,39,34])-->(0001,224)`

**采用combine计算电费和**

`mapper1：(0001,34)，(0001,27)，(0001,33)`

`combine:(0001,94)`

`mapper2：(0001,57)，(0001,39)，(0001,34) `    

`combine:(0001,130)`

`reduce：(0001,[94,130])-->(0001,224)`

这两种方法计算都没有问题，但是如果是计算中值，combine就会出错

`mapper1：(0001,34)，(0001,27)，(0001,33) ` 

`combine:(0001,27)`

`mapper2：(0001,57)，(0001,39)，(0001,34) `

`combine:(0001,39)`

`reduce：(0001,[27,39])-->(0001,33)`

而实际值是34

## MapReduce的架构体系

MapReduce是多进程的，优点在于可以充分利用资源，不会出现资源的抢占；缺点是进程慢，适合大批量，时效性要求低，离线的计算

* client：客户端，MapReduce执行的作业的提交入口

* jobtracker：主进程，客户端发送请求给jobtracker，jobtracker分配资源并调度给工作进程tasktracker，通过tasktracker发送的心跳/heartbeat监控其工作状态和任务进度。一旦某个tasktracker工作失败，就把作业交给其他tasktracker。主进程采用进程池的方法处理用户请求和监控，一个MapReduce集群只有一个主进程

* tasktracker：工作进程，每一个节点都只有一个工作进程。每个工作进程隔3秒向主进程发送heartbeat，报告工作进度，如果作业完成，询问是否有新任务

* task：分为map task和reduce task两种，由tasktracker启动

* slot：运行map和reduce的容器

## MapReduce作业的提交

![](.\pictures\MR架构.png)

客户端给jobtracker发送job请求，jobtracker接受，并返回给客户端一个id号。然后客户端在hdfs上建立一个和id号同名的目录，将job内容上传在里面，向jobtracker提交job。jobtracker初始化，接着分配资源并调度各tasktracker工作，同时通过heartbeat和各节点保持通讯，知晓任务进度。tasktracker通过mapper,reducer完成job，作业的代码来源于客户端上传的文件，所用的数据来源于各slave节点

## MapReduce的运行脚本

MapReduce运行脚本需要四个文件和streaming架构

* run.sh文件：包含streaming架构的文件，也是脚本文件

* mapper文件：执行map

* reducer文件：执行reduce

* 数据源文件：MapReduce计算需要的数据

**streaming**：MapReduce是用Java开发的，为了使其更有通用性，streaming框架提供接口，允许使用其他语言实现MapReduce，还可以实现平台的迁移

#### streaming框架

主要包括

* stream_jar_path ：streaming包的路径，Hadoop没有streaming命令，所以要指定stream jar文件
* Hadoop_cmd ：Hadoop解释器路径
* input ：输入文件在hdfs上的路径，是MapReduce的数据源，是map的输入文件，可以是一个，也可以是一个目录。执行前需要预先存储在hdfs上，以便所有节点共享。如果文件有多个，用`,`隔开，或者用正则`*`等通配符匹配
* output：输出文件在hdfs上的路径，该路径一定不能存在，否则会出错，是reduce的输出文件，只有一个。这样设置的目的是防止数据丢失（同名的文件会覆盖，原先的输出文件就会丢失）
* mapper ：执行map的代码文件
* reducer ：执行reduce的代码文件
* file：mapper/reducer文件在本地的路径，把mapper等**本地文件**（不能是目录）临时分发到hdfs上，使所有节点都可以进行map等操作。和输入文件不同的是，该文件不需要提前上传到hdfs，是在运行.sh文件时即时分发的，等到MapReduce任务完成，该文件会自动从hdfs上删除。若本地文件有多个，可以分多次`file localpath`，一般是小文件
* cacheFile：从**hdfs**上分发**文件**给其他节点，必须要写全路径

```bash
cachefile hdfs://master:9000/path(hdfs文件路径)#文件别名
```

* cacheArchive：从**hdfs**上分发**压缩文件**给其他节点，在执行作业时，hdfs会把压缩文件自动解压为一个**目录**。当数据源是目录时，只能先打包，上传至hdfs，然后用cachearchive分发

```bash
cachearchive hdfs://master:9000/path(hdfs打包文件路径)#文件别名
```

map和reduce过程都是通过标准输入流和标准输出流完成的，map过程的输出文件，也就是reduce过程的输入文件，存储在本地磁盘上，一旦作业完成就会被删除。若在reduce任务之前map的输出结果传递给reduce失败，就在另一个节点上重新运行map，重新生成reduce的输入文件

每个MR作业（job）包括输入数据，MR程序和配置信息，都有一个ID，如job_local123456_0001。其中local表示本地运行。每个job都包括map任务（task）和reduce任务，每个task也都有ID号，如job_local123456_0001_m_0000_0表示map task，job_local123456_0001_r_0000_0表示reduce task。终止mr作业：

```shell
hadoop job -kill job_id
```

#### streaming开发优点

* 开发效率高：方便单机调试，在Linux中用streaming方式本地模拟MR过程：

  ```shell
  cat inputfile | mapper | sort | reducer > outfile
  ```

* 执行效率高：用C/C++执行更快

## job执行的优先级

* 先进先出队列（FIFO）

* 指定优先级：从高到低，Very_high-high-normal-low-very_low。优先级高的作业可以凌驾于优先级低的


执行作业的比较顺序：

1.先比较优先级

2.比较map任务添加的时间

3.比较jobid的创建时间

## 提交作业常用的配置

配置文件：-jobconf(1.x版本)/-D(2.x版本)

* 设置输出文件每一行的分割符为逗号

```shell
stream.map.output.field.separator=','  
```

* 设置第一个字段为key

```shell
stream.num.map.output.key.fields=1 
```

* 设置输出文件的partition

```shell
partitioner org.apache.hadoop.mapred.lib.KeyFieldBasedPartitioner
```

* 设置key的第一个字段为partition

```shell
num.key.fields.for.partition ='1' 

# mapred.text.key.partitioner.options="-k1,1"
```

* 设置key的第一个字段起到最后一个为partition

```shell
mapred.text.key.partitioner.options="-k1"
```

* 以key的1，2两个字段为partition

 ```shell
mapred.text.key.partitioner.options="-k1,2"     
 ```

设置key的排序依据

```shell
mapred.output.key.comparator.class=org.apache.hadoop.mapred.lib.KeyFieldBasedComparator 
```

* 以key的第一个字段的数值大小为依据

```shell
mapred.text.key.comparator.options="-k1,1n"
```

以key的第一个字段的数值大小逆序排列，再以key的第二个到第三个字段，按文本序升序排序

```shell
mapred.text.key.comparator.options="-k1,1nr -k2,3" 
```

常见参数：

|                参数                 |                含义                |
| :---------------------------------: | :--------------------------------: |
|          mapred.map.tasks           |            map task数目            |
|         mapred.reduce.tasks         |          reduce task数目           |
|           mapred.job.name           |               作业名               |
|         mapred.job.priority         |             作业优先级             |
|       mapred.job.map.capacity       |       最多同时运行map任务数        |
|     mapred.job.reduce.capacity      |      最多同时运行reduce任务数      |
|         mapred.task.timeout         | 任务没有响应（输入输出）的最大时间 |
|     mapred.compress.map.output      |         map 的输出是否压缩         |
| mapred.map.output.compression.codec |         map 的输出压缩方式         |
|       mapred.output.compress        |       reduce 的输出是否压缩        |
|   mapred.output.compression.codec   |       reduce 的输出压缩方式        |



  

