flume是一个用于数据采集的分布式的系统，可从多种不同的数据源收集和移动大规模日志信息到一个数据存储中心（如 HDFS 、HBase）。flume可以在日志系统中定制各类数据发送方，用于收集数据，对数据进行简单处理，写到各种可定制的数据接收方

flume的数据源可以是Console 、RPC 、Text 、Tail 、Syslog 、Exec 等，支持水平扩展（多个agent串联）、垂直扩展（多个agent并联）

# 架构

![](.\pictures\架构.png)

flume由agent和collecter组成，两者本质一样，作用不同。前者负责信息的传递，后者负责信息的收集

## agent

一个flume有一个或多个agent，每一个agent都是一个独立的守护进程，agent的组成：

![](.\pictures\agent.png)

### 三个必要组件

**source：**数据输入，当一个 flume 源接收到一个event时，将通过一个或者多个channel存储该event

**channel：**存储池，采用被动存储的形式，会缓存该event直到该event被 sink组件处理，当数据处理后才会在channel中删除（单次跳转可靠性）。channel 是一个完整的事务，这一点保证了数据在收发时的一致性 、event交互的可靠性，并且它可以和任意数量的 source和 sink 链接

channel保证flume的可靠性的方式：

* memory channel：数据存在内存里，读取快，但有丢失风险

* file channel：数据存在磁盘里，读取慢，但更安全，数据不丢失 （通过WAL实现）

数据采用批量处理，当出现节点异常，这一批数据都不会写入，上个节点重新发送数据；数据可暂存，当目标不可访问后，数据会暂存在 Channel 中，等目标可访问之后，再进行传输

**sink：**数据输出，将event从 Channel 中移除，并将event放置到外部数据介质上，分为：

* 存储event到最终目的的终端：HDFS 、Hbase

* 自动消耗：Null Sink

* 用于agent之间通信： Avro

Source 和 Sink 封装在一个事务的存储和检索中，即event的放置或者提供由一个事务通过channel来分别提供。这保证了事件集在流中可靠地进行端到端的传递：

* Sink 开启事务
* Sink 从 Channel 中获取数据
* Sink 把数据传给另一个 Flume Agent 的 Source 中
* Source 开启事务
* Source 把数据传给 Channel
* Source 关闭事务
* Sink 关闭事务

### 两个可选组件

**Interceptor：**拦截器，不是过滤的作用，而是提取信息

可分为：

* Timestamp Interceptor：在event的header中添加一个 key- timestamp，value 为当前的时间戳

* Host Interceptor：在 event 的 header 中添加一个 key- host，value为当前机器的hostname或者 ip

* Static Interceptor：可以在 event 的header中添加自定义的 key和value

* Regex Filtering Interceptor：通过正则来清洗或包含匹配的 events

* Regex Extractor Interceptor：通过正则表达式来在 header 中添加指定的 key，value 为正则匹配的部分

flume的拦截器也是chain形式的，可以对一个source指定多个拦截器，按先后顺序依次处理

**selectors：**选择器，起路由选择的作用，有两种类型：

* Replicating Channel Selector （复制）：是默认的类型，将 source 传递的 events 发往所有 channel

* Multiplexing Channel Selector（复用）：可以选择events发往哪些 channel 

Interceptor在source之前，selectors在source之后

# event

flume的数据传输的基本单元是event，使用event对象作为数据传递的格式。event由header和body组成，header是key/value形式的，是可选的，起路由选择的作用；body是一个字节数组，存储的是实际内容，是必有的

