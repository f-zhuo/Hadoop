storm是一个开源实时计算框架，特点有：

* 支持多种编程语言：thrift server可以实现

* 存在一个本地模式，用于测试代码

* 可靠性高：无论数据是否得到处理都会反馈（ack机制），若没有处理，重新发送数据

* 有底层消息队列，处理更高效

* 支持本地模式，可模拟集群所有功能

# MR VS storm

和MapReduce对比，storm面向实时处理，MapReduce面向离线

## Mapreduce

全量处理，批处理，一次性同时处理整个数据集

优点：多进程，稳定、吞吐能力强

缺点：时效性差

## Storm

增量式处理，数据来多少处理多少，可以实现实时的数据分析

优点：时效性强，毫秒级别，没有持久化层，计算更快

缺点：增量式处理，没有结束状态，除非手动杀死进程；多线程，一个线程出错整个进程结束；吞吐能力不及MR

*持久化：把数据从内存写入磁盘存储，storm没有持久化，所以storm的输入一般是NoSQL，从内存中读取，或者从消息队列比如Kafka读取*

# storm架构

![](.\pictures\storm.png)

* Nimbus：主节点，负责资源分配，任务调度和监控，类似MR1里的 JobTracker，负责在集群里面分发代码，分配计算任务给Supervisor，并且监控子节点的状态

* Supervisor：从节点，负责接收 nimbus 分配的任务。每个工作节点只有一个supervisor，启动、停止和监控自身管理的 worker 进程（每一个工作进程执行一个 Topology 的一个子集，一个 Topology 由运行在很多节点上的不同 worker 工作进程组成）

* zookeeper：storm所有的状态信息都是存储在zookeeper里的。nimbus通过在zookeeper里写入状态信息分配任务；supervisor通过读取zookeeper的状态信息执行任务，同时通过心跳和zookeeper建立联系，告知nimbus自身状态

  首先启动Zookeeper才能启动storm，Nimbus 和Supervisor之间的所有协调工作都是通过Zookeeper 集群完成。Nimbus和Supervisor进程都是快速失败（ fail fast ）和无状态的：storm是不存储数据的，状态要么在 Zookeeper 里，要么在本地磁盘上，所以重启Nimbus 和 Supervisor 进程， 任务可以继续提交，就好像什么都没有发生过

*worker是运行具体处理组件逻辑进程，包括多个executor，一个executor可以执行一个（默认）或多个task（可设置），是真正的线程，执行的task就是spout或者bolt*

![](.\pictures\worker_process.png)

# storm计算逻辑

storm的基本数据单元是tuple，stream就是由tuple组成的有向无界的数据流。tuple中包含了各种数据类型，也可以自定义数据类型

## 网络拓扑

topology，计算逻辑的封装，可以理解为一个job，由spouts和bolts组成的

storm的原语包括spouts和bolts，由stream grouping连接。所谓原语，类似于MR的map和reduce，每一个原语都是一个线程，stream grouping类似于MR的partition

Topology的定义是一个Thrift 结构，并且Nimbus就是一个Thrift 服务，可以提交由其他语言创建的 topology

### spouts

spouts是消息来源，消息生产者。如果消息没有被成功处理，可重新发送一个 tuple。对接的数据源可以是Hbase、Kafka，可指定发送多个stream流，一个topology只有一个spouts，在开始位置

### bolts

bolts是消息处理逻辑，如过滤，访问数据库，数据格式化，聚合。可以发射多个stream流，主方法为 execute：以 tuple 为输入，处理具体的 tuple。一个topology有多个bolts，在中间和结束位置

![](.\pictures\topology.png)

### Topology流程

以分词为例，第一个spouts对输入的语句进行分词，将分开的单词随机输出给blots，blots将所有单词计数为1，按Stream Grouping分组输出给下一个blots，该blots对单词进行归并计数。因为storm是增量式处理，所以count不可能是全局的，只能是当前数据中的总和

### Stream Grouping

定义怎么从spout/bolt 发射 tuple 到另外的bolt。可以调用 TopologyBuilder 类的 setSpout 和 setBolt 来设置并行度，即设置 task的数量。Stream Grouping常见的有四种：

* Shuffle Grouping ：随机分组，负载均衡

* Fields Grouping： 按指定的 field 分组

* All Grouping ：广播分组，每个输出都要分配一份给下一轮的bolt

* Global Grouping ：全局分组，所有输出分配给一个bolt

![](.\pictures\stream_grouping.png)



Topology任务的提交：

```shell
Storm jar code.jar MyTopology [arg1 arg2]
```

* storm jar 负责连接到 Nimbus 并且上传 jar 包
* 运行主类 MyTopology , 参数是 arg1, arg2 。这个类的 main 函数定义这个 topology 并且把它提
  交给 Nimbus
* Nimbus 通过 Thrift 服务执行 topology

# 应用场景

## 流式处理

最常用的应用场景

* 过滤：对输入按条件过滤

* 组装：storm每次增量处理的数据量有限，超过这个限度就可能造成阻塞，重启可以恢复正常；但因为storm是不持久化的，若输入来源是消息队列，阻塞期间的输入就会丢失

  解决方法：

  * 发生阻塞时通知消息队列禁止发送消息
  * 使用hbase作为输入，hbase有版本号，可以根据时间做恢复
  * 使用一个bolt，记录所有其他原语的关键信息并存在hbase里，这种模式称为**组装**

* join：若待join的两个结果有一个先执行完毕，就在内存中等待，等待时间过长可能会影响内存

## 持续计算

比如反馈系统，storm的输出还可以作为其输入

## 分布式RPC

client-server架构，获取client请求，用topology计算，输出给server

# 容错

## 架构容错

* Zookeeper或本地磁盘才存储数据，nimbus和supervisor出错时，直接重启，结果像什么都没发生

* worker和supervisor通过心跳分别告知supervisor和nimbus任务的进度和自身状态

## 数据容错

* timeout：数据传输和处理时间过长，认为超时

* ack机制：保证可靠性，自身也是轻量级线程（特殊task），主要工作是反馈信息和透析，无论数据发送成功/失败，ack都会报告给spout，所有ack成功，任务就成功

  ack会跟踪每一个 spout 发出的 tuple 树，task的个数如果设置太少会影响性能，如果设置太多会浪费空间。ack跟踪tuple树的内存消耗太大，一般不会使用这种方法，而是采用异或

* 用异或的方法，将每一步spout和bolt输出的tupleid和下一步输入bolt的tupleid进行异或，如果数据完全一致，结果就是0，整个过程异或的结果也是0，说明数据被完整传输和正确处理了

监控页面观察：

http://192.168.87.10:8080/index.html
