**分布式文件系统(distributed filesystem)**

把文件分散布置在多台计算机上

**HDFS(Hadoop Distributed FileSystem)**

Hadoop自带的分布式文件系统，有时简称DFS

**HDFS特点**

* 采用流式数据访问
* 大数据
* 时间延迟大
* 存储文件数受到namenode内存大小的限制
* 只能单用户写入，写入操作只能追加在文件末尾

HDFS的基本存储单元叫做block，默认大小是64M（Hadoop1.x）/128M（Hadoop2.x）。文件都是以block的形式存储的，若文件小于64M，则只会占用同等大小的磁盘空间，不会占用64M

block的大小设置的比磁盘块大，是为了最小化寻址时间，这样多个block组成的文件的读写速率就取决于传输速率了

查看文件由哪些block构成
```
hdfs fsck / -files blocks
```

**Hadoop更倾向于存储大文件的原因**

hdfs上存储文件的最小单元是block,但是一个block最多只能存储一个文件，哪怕文件只有1kb，也只能放在一个block里，余下的空间都浪费了。故在占用相同block的前提下，存储大文件更能充分利用内存和空间

# HDFS1.x

## 架构

由三个节点组成：namenode，secondnamenode，datanode

### Namenode

主节点，只有一个，管理文件系统命名空间（文件目录树），存储元数据，比如文件名，权限，组成文件的block等

**namenode存储**：

* 文件名和数据block的映射

* block和实际存储数据的datanode的内存地址的映射

### secondnamenode

是用来保障namenode的，namenode的元数据信息是存储在镜像文件里的，更新信息在编辑日志文件里。secondnamenode会从编辑日志文件更新元数据，把更新的数据写入自身的镜像文件（namenode和secondnamenode各有一个），再把自身的镜像文件写入namenode的镜像文件。当namenode重启时，就会从镜像文件读取更新数据，重启更快。若namenode挂掉，secondnamenode就可以从自身的镜像文件恢复元数据，但可能还是会丢失部分数据

### datanode

子节点，存储和检索数据块

# HDFS2.x

相对于1.x：

* 多出了Resourcemanager（相当于mapreduce的jobtracker，和namenode是同一个机器）和Nodemanager（相当于mapreduce的tasktracker，和datanode同一台机器）
* 2.x的namenode有两个，分别是active namenode和standby namenode
* 2.x架构存在zookeeper

## namenode的容错机制

* 将namenode存储的信息的元数据分散布置在多个文件系统中，挂载网络文件系统（NFS）

* 在另一台节点上，布置一个secondnamenode，定期将日志合并，当namenode挂掉时可以作为新的namenode

## HDFS的容错方法

### namenode federation

在多个节点布置namenode，每个namenode存储一部分的元数据，管理一部分的文件系统，互不影响。但是datanode是共享的。优点就是集群规模可以扩大，可以减小对namenode的内存要求

### namenode HA（高可用）

active namenode和standby namenode存储的内容是一致的：

* 文件名和block的映射

* 命名空间（文件树结构）

运行active namenode和standby namenode的节点硬件配置应该要一致，平时只有active namenode处于active状态，standby namenode处于standby状态，类似于备份。只有当active namenode挂了，standby namenode才会替代active，状态由standby变为active，保证集群不会中断，解决了hdfs1.0的单点故障问题（namenode挂了只能从镜像文件恢复）

#### 保证两个namenode的内容一致的实现

* NFS（共享网络文件系统），存储的是文件名到block的映射

* datanode会同时向两个namenode发送心跳，确保数据一致，存储的是datanode和block位置关系的映射

#### 故障转移控制FC

确保active挂掉，standby可以立即上位：

zookeeper可以管理两个namenode，通过心跳监视其状态。一旦active失效，zookeeper立即感知，并启动standby。为了即时，FC必须和namenode部署在一起。也可以在日常维护时手动执行故障转移，称为平稳的故障转移(graceful failover)

但有时active namenode并没有失效，只是网络连接及响应慢，会被误认为故障，进行了故障转移。此时，active namenode就不能再进行数据处理了，状态不能是active，需要故障规避（fencing）

##### 故障规避的方法有

* SSH杀死namenode进程

* 撤销namenode访问共享存储目录的权限

* 屏蔽网络端口

* 直接对该主机断电

#### QJM（quorum journal manager群体日志管理器）

高可用的一种方式，QJM集群，包括多个journalnode（jn），和datanode部署在一起。通过QJM的投票机制，少数服从多数，确保当datanode存储数据不一致或某个datanode挂掉时，可以选择应用哪一个数据。正是投票这一机制，决定了jn的数量必须是奇数

## 数据完整性校验

在存入和读取数据时都会进行完整性校验

校验方法：

* 校验和

client向namenode存入数据时，client每隔固定的文件大小就会通过CRC32的方法写入一个校验和（数字字母的组合），namenode同步存入数据时，也会对文件进行校验和，一致就接受，不一致就中断

同样，从namenode向client传送数据时也是如此，不过client读取的校验和是上个状态的

* 数据块检测程序DataBlockScanner

会在datanode节点上开启后台线程，检查数据块的完好程度

 ## 可靠性保证

* 心跳：DN向NN报告自身状态

* 数据完整性：crc32

* 空间回收机制

  ​	回收站：./Trash

  ​	直接删除不进入回收站：-skipTrash

* 副本：一份数据默认在3个节点上有3个备份

* secondnamenode

* 快照

  ​	备份，不复制数据，只复制文件元数据，文件名与block的映射关系，用以容错恢复

## ACL（权限控制）

 和linux的权限控制方法一致，分用户，组，其他用户

## 基本操作

hdfs上查看二进制文件

```shell
hadoop fs -text /xxx
```

查看非二进制文件

```shell
hadoop fs -cat /xxx
```

查看文件后几行

```shell
hadoop fs -tail /xxx
```

查看文件前几行

 ```shell
hadoop fs -head /xxx
 ```

创建文件夹

```shell
hadoop fs -mkdir /xxx
```

创建文件

```shell
hadoop fs -touchz /xxx
```

上传

```shell
hadoop fs -put localpath /xxx
```

获取文件

```shell
hadoop fs -get /xxx
```

将xxx目录下的所有文件内容合并到一个文件里（文件内容需要是非压缩）

```shell
hadoop fs -getmerge /xxx
```

剪切

```shell
hadoop fs -mv /xxx /xxx
```

本地文件移动到hdfs

```shell
hadoop fs -moveFromLocal localPath HDFSPath
```

hdfs文件移动到本地

```shell
hadoop fs -moveToLocal HDFSpath localPath 
```

复制

```shell
hadoop fs -cp /xxx /xxx
```

本地文件复制到hdfs

```shell
hadoop fs -copyFromLocal localPath HDFSpath
```

hdfs文件复制到本地

 ```shell
hadoop fs -copyToLocal HDFSpath localPath 
 ```

删除文件

```shell
hadoop fs -rm /xxx
```

删除文件夹

```shell
hadoop fs -rmr /xxx
```

杀死进程

```shell
hadoop job -kill jobid
```





 





 

