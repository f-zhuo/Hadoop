先由一张图直观感受下Hadoop1.x和2.x的区别

![](.\pictures\hadoop2vs1.png)

yarn可以看成是一个分布式的操作系统，可以进行资源整合，把系统资源最大化利用。Hadoop1的结构是以hdfs为文件系统，之上的MapReduce是计算框架，若要进行计算就只能通过MapReduce。Hadoop2在hdfs的基础上增加了yarn，像是磁盘上有了操作系统。通过yarn，计算框架不仅仅只有MapReduce，还可以是spark，storm等，MapReduce变成了一个应用框架，yarn提供了应用接口，这样，同一套硬件集群可以同时运行多个任务

![](.\pictures\yarn.png)

# 架构

Hadoop1只有job tractor和task tractor，Hadoop2舍弃了这两者，变成了Resource Manager，Application Master和Node Manager。job tracker的两个任务-资源管理和作业调度分别被Resource Manager和Application Manager替代

* Resource Manager（RM）：资源管理，实际是通过一个纯粹的调度器scheduler进行资源分配，部署在主节点，代替了集群管理器

* Application Manager（AM）：作业调度，部署在从节点，一个从节点可以有多个AM。实际上am也是一个container

* Node Manager(nM)：部署在每个从节点上，管理自身资源，接受RM的请求和AM的调度，分配container资源，通过心跳告诉RM自己的状态

* container：是一个进程，由NM启动，是真正运行任务的地方

*Hadoop1.0的job在2.0对应着application*

Hadoop1.0资源运行任务的地方是slot，包含的资源是CPU和内存，是绝对平均分配的，比如资源一共有4CPU，32G内存，slot有2个，那么每个slot都有2CPU，16G内存，容易造成资源利用率过高或过低。而container可以调整最小容量来改变其数量，使资源利用合理；Hadoop1的slot区分map和reduce，Hadoop2不区分

![](.\pictures\架构.png)

# 运行过程

![](.\pictures\yarnyu.png)

- Client 请求 ，Resource Manager 选择一个 Node Manager ，启动一个Container 并运行 Application Master 
- Application Master 根据实际需要向 Resource Manager 请求更多的资源，调度更多Node Manager参与，NM会启动自身的container
- Application Master 通过获取到的 Container 资源执行分布式计算

# 容错能力

* RM挂掉：进行主备替换，基于ZooKeeper的高可用，会有备份替换当前的RM

* AM挂掉：RM会重启它，但之前的任务已经被保存，不需要再运行一次

* NM挂掉：以心跳通知RM，进一步处理。如果NM上没有AM，整个任务不会挂；如果有，整个任务就挂掉

# yarn的调度器

* FIFO scheduler：按提交顺序执行，如有小任务在大任务之后，必须等待大任务完成，效率低

* Capacity scheduler：多队列并行，会有专有队列运行小任务，预先占用一部分资源，如有小任务和大任务并行，效率提高；但如果只有一个任务，会有队列空运行，浪费资源

* Fair scheduler：动态调整，大任务运行中遇到小任务会分出一部分资源让小任务运行