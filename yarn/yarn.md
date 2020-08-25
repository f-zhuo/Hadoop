先由一张图直观感受下Hadoop1.x和2.x的区别

![](.\pictures\hadoop2vs1.png)

yarn可以看成是一个分布式的操作系统，可以进行资源整合，把系统资源最大化利用。Hadoop1.x的结构是以hdfs为文件系统，之上的MapReduce是计算框架，若要进行计算就只能通过MapReduce。Hadoop2.x在hdfs的基础上增加了yarn，像是磁盘上有了操作系统。通过yarn，计算框架不仅仅只有MapReduce，还可以是spark，storm等，MapReduce变成了一个应用框架，yarn提供了应用接口，这样，同一套硬件集群可以同时运行多个任务

![](.\pictures\yarn.png)

# 架构

Hadoop1.x只有job tractor和task tractor，Hadoop2.x舍弃了这两者，变成了ResourceManager，ApplicationMaster和NodeManager，但依然是master/slave的架构。job tracker的两个任务-资源管理和作业调度分别被Resource Manager和ApplicationMaster替代

## Resource Manager（RM）

全局资源管理器，是master，只有一个。负责资源的管理和分配，包括Applications Manager（ASM）和scheduler（调度器）。前者负责资源的管理，后者负责资源的分配，部署在主节点

### scheduler

只是负责资源的调度和分配，将资源集中在container里，不管其他任何事情，哪怕任务失败，也与其无关。scheduler是可插拔的，可根据不同需要选择不同的类型，一般有三种：

* FIFO scheduler：按提交顺序执行，如有小任务在大任务之后，必须等待大任务完成，效率低

* Capacity scheduler：多队列并行，会有专有队列运行小任务，预先占用一部分资源，如有小任务和大任务并行，效率提高；但如果只有一个任务，会有队列空运行，浪费资源

* Fair scheduler：动态调整，大任务运行中遇到小任务会分出一部分资源让小任务运行

### Applications Manager（ASM）

应用程序管理器，负责管理所有的应用程序，分配资源。和Application Master、scheduler协商资源，提交应用程序，在AM失败时重启它

## Application Master（AM）

和RM协商获取资源，调度作业，将任务进一步分配给NM，和NM通过心跳建立联系，监视NM状态，当有任务失败时就重新申请资源并重启任务。每一个提交的作业都有一个AM，部署在从节点，一个从节点可以有多个AM，实际上AM也是一个container

### Node Manager(NM)

是slave，有多个，部署在每个从节点上，管理自身的资源和任务。会通过心跳向RM报告自身资源的使用情况，接受AM的任务指令，分配给自身container资源以启动任务

### container

是一个进程，由NM启动，是真正运行任务的地方，是一个资源的抽象，包含内存，磁盘，CPU，网络等

*Hadoop1.0的job在2.0对应着application*

Hadoop1.0资源运行任务的地方是slot，包含的资源是CPU和内存，是绝对平均分配的且区分map和reduce slot，资源无法共享。比如资源一共有4CPU，32G内存，slot有2个，那么每个slot都有2CPU，16G内存，容易造成资源利用率过低；当只有map task/reduce task，另一个slot的资源浪费。而container可以调整最小容量来改变其数量，使资源利用合理

![](.\pictures\架构.png)

# 运行过程

![](.\pictures\yarnyu.png)

- Client 请求 ，ResourceManager 选择一个 NodeManager ，启动一个Container 并运行 ApplicationMaster 
- ApplicationMaster 根据实际需要向 ResourceManager 协商请求更多的资源，将任务分配，调度更多NodeManager参与，NM会启动自身的container
- ApplicationMaster 通过获取到的 Container 资源执行分布式计算

# 容错能力

* RM挂掉：进行主备替换，基于ZooKeeper的高可用，会有备份替换当前的RM

* AM挂掉：RM的ASM会重启它，但之前的任务已经被保存，不需要再运行一次

* NM挂掉：以心跳通知RM，进一步处理。如果NM上没有AM，整个任务不会挂；如果有，整个任务就挂掉

