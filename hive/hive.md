## hive和hbase区别

* 数据库主要用于事务交易，存储空间紧密，没有冗余数据，对读和写都有优化

- 数据仓库主要用于数据分析，存储海量数据，有冗余，只对读有优化

**Hive：**是基于Hadoop的数据仓库，sql解析引擎，MR的封装，把sql转成MR任务。hive的表是逻辑表，只有表定义，即元数据，表现为Hadoop的目录/文件。不存数据，数据存储在hdfs上，达到了元数据与数据存储分离的目的。读多写少，默认不支持数据的更新和删除，可以手动设置支持（新版本）

**Hbase**：数据库，hdfs的封装

## 数据格式

数据存储在文件和记录的规则

Hive没有定义数据格式，由用户自定义，需要指定三条属性：

* 列分隔符：空格、\t、\001

* 行分隔符：\n

* 文件读取方法：
  * TextFile（默认格式，存储方式是行存储，数据不压缩）
  * SequenceFile（二进制文件，可压缩）
  * RCFile（hive独有，行列存储相结合）

自定义数据格式时，10可以存储为string型，依然可以进行算术运算，但排序时会按照字符串的字节序，比如string型的10排在9前面（升序）

## HQL和SQL区别

![](.\pictures\HQL&SQL.png)

**HQL拓展性**

自定义函数

|                      函数名                       |        含义        |                             描述                             |
| :-----------------------------------------------: | :----------------: | :----------------------------------------------------------: |
|        UDF <br />(user-defined   function)        | 用户自定义普通函数 | 输入输出一对一，直接应用于select语句，例如查询时需要对字段做格式化处理（如大小写转换） |
| UDAF<br />(User- Defined   Aggregation Funcation) |      聚合函数      |                  输入输出多对一，如group by                  |
|     YDTF<br />(User-Defined Table Functions)      |      生成函数      |                 输入输出一对多，比如explode                  |

**读时模式（HQL）**：只有hive读的时候才会检查，解析字段和schema，写的时候不检查，数据加载快

**写时模式（SQL）**：写的慢，需要建立索引，压缩，数据一致性，字段检查， 读的时候速度快

 ## hive体系

* 客户端：client

* 语句转换器：driver，是hive核心，把sql语句转换成相应的任务

* hdfs：数据存储

  元数据（metastore）存储：
  * 本地：单用户模式（默认derby数据库），多用户模式（MySQL数据库）

  * 远程：多用户模式（MySQL数据库）

## Hive的数据模型

hive 默认表存放路径一般都是在工作目录的 hive 目录里面，按表名做文件夹分开，如果有分区表，分区值是子文件夹

### table

* 内表：导入数据，数据会移动到数据仓库目录下；表格删除，数据和元数据也删除

` create table`

* 外表：导入数据，数据不会移动到数据仓库目录下；表格删除，数据不会删除，只删除元数据

`create external table`

### partition

在 Hive 中，表中的一个 Partition 对应于表下的一个目录，所有的 Partition 的数据都存储在对应的目录中

例如： info 表中包含 dt 和 city 两个 Partition ，则对应于 dt = 20090801, city = shanghai 的 HDFS 子目录为： /info/dt=20090801/city =shanghai

设置partition的目的是辅助查询，缩小查询范围

做分区的字段：经常查询但分组有限的字段

### bucket

**分库**：把一张表拆分成多个表

hive通过哈希的方法进行分库

table 和 partition 可以通过 CLUSTERED BY进一步分 bucket

```shell
create table bucket_user (id int,name string) clustered by (id) into 4 buckets;
```

bucket的数量和hive过程的reduce数量一致

bucket的意义在于

* 优化查询，分库后的表进行join时数据量变小，会自动激活map端的map-side join

* 方便采样

 #### 采样

语法：` TABLESAMPLE(BUCKET x OUT OF y on field)`

y可以理解为桶的切分大小，必须是bucket的倍数或者因子；x是采样开始的桶号。

例如，table 总 bucket 数为32时，` select * from student tablesample (bucket 3 out of 8)`，表示总共抽取 32/8=4 个 bucket 的数据，分别为第 3 个 、第（ 3+8=）11 个、第（3+8+8=）19个和第（3+8+8+8=）27个 bucket（文件存储为part） 的数据

 ## 数据类型

### 原生类型

* Tinyint

* Smallint

* Int

* bigint

* Float

* Double

* Boolean

* String

* Date

* Binary（Hive 0.8.0以上才可用）

* TimeStamp（Hive 0.8.0以上才可用）

### 复合类型

* array：有序字段，字段类型必须一致，`ARRAY< data_type>`

* Map：无序键值对，键的类型必须一致，值的类型也必须一致，`MAP<primitive_type,data_type>`

* Struct：一组命名字段，字段类型可以不同，`STRUCT <col_name:data_type [COMMENT col_comment],…>`
  * union：可以指定里面包含的任意一种类型 `UNIONTYPE<data_type,data_type…>`如 `u UNIONTYPE<int,float,map<string,int>,stuct<a:int,b:string>>`可以用数字指代使用的类型，如前例中0代表int，1代表float，2代表map，3代表struct

 ## hive优化

### map优化

控制map个数，变相控制block size

* 设置每个map处理的最大的文件大小，单位为B，从而影响启动多少个map数

```shell
set mapred.max.split.size=256000000;
```

* 设置节点中可以处理的最小的文件大小

```shell
set mapred.min.split.size.per.node=1;
```

* 设置机架中可以处理的最小的文件大小

```shell
set mapred.min.split.size.per.rack=1;
```

* 设置map的个数

```shell
set mapred.map.tasks =10;
```

* 在map端做聚合，也就是combiner

```shell
hive.map.aggr=true 
```

### reduce优化

设置reduce任务处理的数据量

```shell
hive.exec.reducers.bytes.per.reducer
```

控制reduce个数

* 设置reduce个数

```shell
set mapred.reduce.tasks=10
```

* group by

  * 如果没有group by，只有一个reduce

  * group by时，相同的key是一个reduce处理，key比较多的就处理的慢，负载不均衡

     解决方法：在reduce聚合之前，预聚合一次，但预聚合不按key，而是随机分配。目的是防止某个key的数据过多，造成某个reducer的压力过大。预聚合使数据量变小，负载均衡

```shell
hive.groupby.skewindata=true
```

* order by：对指定字段进行排序，是全局排序，所以只有一个reduce（多个reduce只能保证局部有序，无法保证全局）。当数据量过大时，一个reduce处理性能不佳

将order by用distribute by和sort by结合替换，产生多个reduce

**distribute by**：类似于mapreduce中的partition，按字段将map端输出分配给不同的reduce

```shell
select * from t distribute by id;
```

**sort by**：和order by的区别在于，sort by不是全局排序，是在reduce之前做的排序，相当于在各partition中排序

```shell
select * from t distribute by itemid sort by userid desc;
```

**cluster by**：相当于distribute by和sort by结合，默认只能倒序，不可更改，而且partition和sort的字段只能一致

```shell
select * from t distribute by itemid sort by itemid;

select * from t cluster by itemid；
```

### join过程优化

做笛卡儿积时连接条件的判断比较耗时，join用on不用where，避免过程出现大量结果

如果join过程中不加on或者无效on，会使用一个reduce

 ### count distinct

数据有大量空值

方法：先过滤

```shell
select cast(count(distinct(user_id))+1 as bigint ) as user_cnt
from t
where user_id is not null and user_id <>'';
```

### union all

union去重，union all不去重，性能更优

### multi-insert

从一个表向多个表插入数据，可以将多个插入语句写在一起

```shell
from a
insert table b select user_id
insert table c select user_name;
```

### automatic merge

将多个小文件合并

启动一个独立的map-reduce任务来进行文件merge

```shell
hive.merge.size.per.task = 256 * 1024 *1024
```

单位字节

### 设置并行

```shell
set hive.exec.paralled = true
```

### 数据倾斜

表现：

* 任务进度长时间维持在99%（或100%），查看任务监控页面，发现只有少量（1个或几个）reduce子任务未完成

* 查看未完成的子任务，可以看到本地读写数据量积累非常大

可能引起数据倾斜的操作：

* Join

* Group by 

* Count Distinct 

原因：

* key分布不均 

* 业务数据特点 

#### 常见的数据倾斜

* 大小表关联

join时默认左边的表是小表，会放入内存。小表放内存使reduce运行效率更高。但如果小表的key比较集中，读入内存的数据也会变多，不适合放入内存

指定放内存的表：`/*+MAPJOIN(table_name)*/`

```shell
select /*+MAPJOIN(a)*/ * from a join b on a.user_id =b.user_id
```

指定大表，尽量不存内存：`/*+STREAMTABLE(table_name) */ `

```shell
select /*+STREAMTABLE(b)*/ * from a join b on a.user_id =b.user_id
```

* 大大表关联

部分的 userid 是空或者是 0 ，导致在用 userid 进行 hash 分桶的时候，会将 userid 为 0 或者空的数据分到一起，导致了过大的数据倾斜 

解决方法：

* 把空值的 key 变成一个字符串加上随机数，把倾斜的数据分到不同的 reduce 上，由于 null 值关联不上，处理后并不影响最终结果 

以a的uid和b的user_id做连接条件

```shell
on case when ( a.uid = '0 or a.uid is null) then concat('a.uid',rand ()) else a.uid end = b.user_id
```

* 两个表数据量都很大，可以先过滤有用的那一些记录

```shell
select * from log join user on log.user_id =user.user_id
```

优化

```shell
select /*+MAPJOIN(d1)*/ *
from log t1
join (
select/*+MAPJOIN(t)*/ d.*
from (
select user_id from log group by user_id
) t
join user d
on t.user_id =d.user_id
) d1
on t1.user_id=d1.user_id;
```



