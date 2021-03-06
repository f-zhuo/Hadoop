# 定义

概括来说，zookeeper是一个松散耦合的分布式系统的粗粒度锁以及可靠性存储（低容量）的系统

* 松散耦合：对硬件要求不严格，在大部分节点都能使用

* 分布式：多节点

* 粗粒度锁：粒度是节点级的，各节点在物理隔离的状态下，通过这把锁来维持生态

* 可靠性存储：每个节点都可以存储数据，一般是配置文件和共享资源，但存储容量小

Zookeeper是一个分布式锁，保证数据的一致性，对集群的稳定性起到关键作用，在任何一个节点都可以访问

# 数据模型

类似于文件管理系统，每个“目录文件”都叫做节点，可以存储数据（数据量少），访问节点必须要指定绝对路径，不能是相对路径。每个节点的信息包括数据，数据长度，创建时间，修改时间等，节点可以创建、删除、修改

节点分为四类：

* persistent nodes

永久节点，除非手动删除，否则一直存在。创建时的默认节点

* ephemeral nodes

临时节点，由client创建，在client连接时有效，client失效，该节点会被zookeeper删除

*  persistent_sequential  nodes

永久顺序节点，和永久节点一样，只是zookeeper会按顺序给节点末尾追加序列号

* ephemeral_sequential  nodes临时顺序编号目录节点 

临时顺序节点，和临时节点一样，只是zookeeper会按顺序给节点末尾追加序列号

后两种节点常用于分布式系统的节点命名和选举主节点上

从选举的角度，节点又可以分为：

* leader：相当于主节点，提供读写服务
* follower：相当于从节点，提供读服务，参与选举
* observer：是一种特殊的follower，提供读服务，不参与选举

zookeeper有三种运行模式： 集群模式、单机模式和伪集群模式，伪集群模式：在单机上部署多个zookeeper，模拟集群 

# zookeeper主从一致的实现

基于Zab（zookeeper atomic broadcast/原子广播）实现，Zab分为恢复模式（选主）和广播模式（同步）。当集群刚刚启动或者leader失效时，就会启动恢复模式选出leader，然后通过广播模式使得follower和leader一致，当大多数follower同步结束后恢复模式就结束了

遵守的是paxos协议

## paxos

一种一致性算法，分成两个角色，proposer和acceptor

* proposer向大多数的acceptor发起提案，acceptor接收后会和之前接收的提案进行比较，若提案编号比之前大，acceptor会回复proposer，同时把之前提案的回复内容也一并发送。对于编号小于之前的提案，acceptor不会回复
* proposer接收大部分acceptor的回复后会请求accept，acceptor接收后，在不违反自身对其他proposer的承诺的前提下，会同意提案

## ZAB

存在两个模式

* 广播模式：数据同步
  * 客户端发起请求后，leader会把请求变成若干个proposer，每个proposer都有一个唯一的ID
  * leader和每个follower都有一个消息队列，leader把消息放在队列里，follower从队列取出消息，同时发送ACK给leader
  * leader接到半数以上的ACK就会发送commit
* 恢复模式：选主

# 监控机制

常用的三个命令

* getData：监控数据
* exists：监控节点是否存在
* getChildren：监控父节点下的子节点列表

client建立一个watcher事件（监视），如该节点或其子节点数据发生改变，zookeeper就会告知client。若该节点是临时节点，一旦client连接失效，该节点也会自动删除

监控的传递性：子节点数据变化触发监控也会触发父节点的监控

## 监控的风险

* 监控是一次性的，触发后需要再次设置

* 多个事件的监控，有可能只会触发一次

* 客户端有可能看不到所有数据的变化，比如一个客户端设置了关于某个数据点 exists 和 getData 的监控，则当该数据被删除的时候，只会触发“文件被删除”的通知
* 客户端网络中断，无法收到监控的窗口时间，要由模块进行容错设计
* 父节点下的多子节点失效只会触发一次监控

# 访问控制链

每个节点的ACL（access control list）保存了client对节点的访问权限，用三元数组来描述：

(scheme:expression,perms) 即（模式：用户，权限），如 (ip:192.2.0.0/16, READ)

|  模式  |                             描述                             |
| :----: | :----------------------------------------------------------: |
| world  | 只有一个用户anyone, world:anyone代表任何人，是所有人有权限的结点 |
|  auth  |                        已经认证的用户                        |
| digest |        通过username：password字符串的MD5编码认证用户         |
|  host  | 匹配主机名后缀，如host:com匹配host:host1.com，但不能匹配host:host1.cn |
|   ip   |            通过IP识别用户，表达式格式为 addr/bits            |

|  权限  |           描述           |
| :----: | :----------------------: |
| create |        创建子节点        |
|  read  | 读取节点数据和子节点列表 |
| write  |       修改节点数据       |
| delete |        删除子节点        |
| admin  |       设置节点权限       |

# Zookeeper应用场景

## 配置管理

用永久节点实现全局的系统配置：在一个server上设置zookeeper永久节点，该节点存储有配置文件，其他节点都可以访问，完成全局的配置

## 集群管理

用临时节点实现：client每连接一个server时，会申请创建一个临时节点，同时父节点和client建立监视。若某个server下线，client连接失效，对应的节点自动删除

## 选主

用临时顺序节点实现：在集群中，选择当前序列号最小的一个节点作为主节点。若主节点对应的server挂掉，主节点删除，再选取下一个最小的做为新的主节点。此时原主节点恢复，也只能获得更大的序列号

## 分布锁

使用临时顺序节点实现：首先建立一个锁的根目录，每个客户端连接一个server后都在这个根目录下创建临时顺序节点，从1开始。每个客户端获取根目录下的子节点列表，判断与自己连接的节点是否是当前序列号最小的，若是则获得锁，执行业务，结束后释放锁。否则就监听自己前一位的子节点的删除消息，直至获得锁

## 队列管理

* 同步队列：当队列所有成员都聚齐才会开始工作，用临时顺序节点实现

首先创建一个父目录，每个队列成员连接时都会创建临时顺序节点，同时监控标志位目录是否存在。若存在，就工作。每个队列成员还会获得父目录的所有节点，判断个数是否等于队列大小，若等于就创建标志位目录

* FIFO：先入先出队列，先入队的节点先工作

先创建一个父目录，每个队列成员连接时都会创建临时顺序节点，选择当前序列号最小的那个工作，结束就删除该节点，继续选择下一个最小的

# zookeeper特性

序列一致性：客户端发送的多个更新请求是按照同样的顺序在 Zookeeper 进行更新

原子一致性：更新只能成功或者失败，没有中间状态

单系统镜像：无论连接哪台 Zookeeper 服务器，客户端看到的服务器数据一致

可靠性：任何一个更新成功后都会持续生效，直到另一个更新将它覆盖

实时性： Zookeeper 保证所有客户端将在一个时间间隔范围内获得服务器的更新或者失效信息。但由于网络延时等原因， Zookeeper 不能保证两个客户端能同时得到刚更新的数据，如果需要最新数据，应该在读数据之前调用sync() 接口

# 常用命令

* 启动

在bin目录下

```shell
./zkServer.sh start
```

zookeeper的启动需要在每个节点执行，不像Hadoop只在主节点启动

* 检验zookeeper是否启动

在bin目录下

```shell
./zkServer.sh status
```

结果是leader/follower

* 停止

```bash
./zkServer.sh stop
```

* 执行客户端

```bash
zkCli.sh
```

* 连接zookeeper

```bash
zkCli.sh -server localhost:2181
```

* 查看当前 ZooKeeper 中所包含的内容

```bash
ls / 
```

* 创建节点

```bash
create /text "test" 
create -e /text "test" # 创建临时节点
create -s /text "test" # 创建序列节点
```

* 设置节点内容为123

```bash
set /test 123
```

* 查看节点

```bash
get /test
```

* 删除节点

```bash
delete /test
```

* 删除节点目录

```bash
rmr /home/test
```
