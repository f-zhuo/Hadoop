# Hive和Hbase区别

**Hive：**是基于Hadoop的数据仓库，可以把SQL语句转成MR任务。Hive的表是逻辑表，只有表定义，只存储元数据，表现为Hadoop的目录/文件；不存数据，数据存储在HDFS上，达到了元数据与数据存储分离的目的。读多写少，存储海量数据，有冗余，只对读有优化，默认不支持数据的更新，可以手动设置支持（新版本）

**Hbase**：Hadoop database的简称，是一种NOSQL数据库，是HDFS的封装，没有冗余数据，对读和写都有优化

**MySQL**：关系型数据库，数据都是结构化的，强调强一致性

# 数据类型

## 原生类型

* tinyint
* smallint
* int
* bigint
* float
* double
* decimal
* boolean
* string
* char
* varchar
* date
* binary（Hive 0.8.0以上才可用）
* timestamp（Hive 0.8.0以上才可用）

## 复合类型

* array：类型一致的有序字段的集合，`ARRAY<data_type>`

* map：无序键值对，键的类型必须一致且属于原生类型，值的类型也必须一致但可以是任意类型，`MAP<primitive_type,data_type>`

* struct：字段的集合，但字段类型可以不同，`STRUCT<col_name1:data_type1,…>`
  * union：可以指定里面包含的任意一种类型 `UNIONTYPE<data_type,data_type…>`如 `u UNIONTYPE<int,float,map<string,int>,stuct<a:int,b:string>>`可以用数字指代使用的类型，如前例中0代表int，1代表float，2代表map，3代表struct

定义数据类型时，10存储为string型，依然可以进行算术运算，但排序时会按照字符串的字节序，比如string型的10排在9前面（升序）

# 数据格式

数据格式：数据存储在文件和记录的规则

Hive没有定义数据格式，由用户自定义，需要指定三条属性：

* 列分隔符：\001或^A(ctrl+A)、\002或^B(ctrl+B)、\003或^C(ctrl+C)

* 行分隔符：\n

* 存储格式：
  * TextFile：默认格式，存储方式是行存储，数据不压缩
  * SequenceFile：二进制文件，可压缩
  * RCFile：hive独有，行列存储相结合
  * ORCFile：RCFile的升级版，将数据按行分块，块按列存储，可以压缩

# 自定义函数

HQL的拓展，有三种，可以直接应用于select语句：

* UDF (user-defined  function)：用户自定义函数，输入输出一对一，例如对字段做格式化处理（大小写转换等）
* UDAF(User- Defined  Aggregation Funcation)：用户自定义聚合函数，输入输出多对一，如group by
* UDTF(User-Defined Table Functions) ：用户自定义表生成函数，输入输出一对多，比如explode

# HQL和SQL区别

![](.\pictures\HQL&SQL.png)

**读时模式**：只有读的时候才会检查，解析字段和schema，写的时候不检查，数据加载快

**写时模式**：写的慢，需要建立索引，压缩，保证数据一致性，字段检查， 读的时候速度快

# Hive体系

## 客户端

client，发起查询请求

## 语句转换器

driver，是hive核心，把SQL语句转换成MR的任务

### Hive转成MR过程

* 对SQL进行词法和语法解析，生成抽象语法树（AST）
* 遍历AST，生成SQL的基本组成单元QueryBlock：AST比较复杂，不够结构化，不能直接翻译为MR任务。QueryBlock分为三个部分：输入，计算和输出，可以简单理解为子查询
* 遍历QueryBlock，生成执行操作树operator tree：MR的map任务和reduce任务都是由operator完成的，操作符包括SelectOperator，JoinOperator等
* 使用逻辑层优化器对operator tree优化，比如合并操作符，减少shuffle和reduce的数量，生成新的operator tree
* 遍历operator tree，生成MR任务
* 使用物理层优化器对MR任务进行优化处理，比如mapjoin

## HDFS

数据存储的地方

hive元数据（metastore）存储有三种模式：

* 内嵌derby：是hive的默认启动模式，同一时间只能连接一个数据库，属于单用户模式
* local：本地模式，可以使用MySQL本地连接，属于单用户模式
* remote：远程模式，可以使用MySQL远程连接，属于多用户模式

# Hive的数据模型

hive 默认表存放路径一般都是在工作目录的 hive 目录里面，按表名做目录分开，如果有分区表，分区值是子目录

## table

分为内表和外表

* 内表：导入数据，数据会移动到数据仓库目录下；表格删除，数据和元数据也删除

```sql
create table
```

* 外表：导入数据，数据不会移动到数据仓库目录下；表格删除，数据不会删除，只删除元数据。只要重建表结构就能恢复表

```sql
create external table
```

## partition

分区表，在 Hive 中，表中的一个 partition 对应表的一个子目录，所有的 partition 的数据都存储在对应的目录中

例如： info 表中包含 dt 和 city 两个 partition ，则对应于 `dt = 20090801, city = shanghai `的 HDFS 子目录为：` /info/dt=20090801/city =shanghai`

partition的目的是加快查询速度，缩小查询范围，避免全表扫描

适合做分区的字段：经常查询且分组有限的字段

## bucket

分桶，在表或者分区表的基础上再进行更细粒度的划分，其实就是一个分库的过程。hive分桶的方法是哈希，bucket的数量和hive过程的reduce数量一致，每个bucket就是表/分区表的一个目录（存储为part）

*分库：把一张表拆分成多个表*

通过 CLUSTERED BY实现 bucket

```shell
create table bucket_user (id int,name string) clustered by (id) into 4 buckets;
```

bucket的意义在于

* 优化查询，bucket后的表进行join时数据量变小，会自动激活map端的map-side join

* 方便采样

**采样**

语法

```sql
TABLESAMPLE(BUCKET x OUT OF y on field)
```

y可以理解为桶的大小，必须是bucket的倍数或者因子；x是采样开始的桶号

例如

```sql
select * from student tablesample (bucket 3 out of 8)
```

当table 的 bucket 数为32时，表示总共抽取 32/8=4 个 bucket 的数据，分别为第 3 个 、第 3+8=11 个、第3+8+8=19个和第3+8+8+8=27个 bucket 的数据

# Hive优化

## 数据倾斜

数据分布不均，有些reduce少，任务早早完成了；有些reduce多，任务迟迟无法完成，导致整个任务无法完成

表现：

* 任务进度长时间维持在99%（或100%），查看任务监控页面，发现只有少量（1个或几个）reduce子任务未完成

* 查看未完成的子任务，可以看到本地读写数据量积累非常大

原因：

* key分布不均 ：key数量多的reduce处理的数据量远大于其他reduce
* 有很多空值或0值：这些空值或0值会直接进入一个reduce
* 数据类型不一致：不一致的数据类型会分配到一个reduce

可能引起数据倾斜的操作：

* join

* group by 

* count distinct 

### join

#### 大小表关联

##### 小表连接

join的一个表是小表，但key比较集中，会分到一个reduce

解决方法：把小表放内存，顺序扫描大表可以使reduce运行效率更高。join时默认左边的表是小表，会放入内存

指定小表放内存：`/*+MAPJOIN(table_name)*/`

```shell
select /*+MAPJOIN(a)*/ * from a join b on a.user_id =b.user_id
```

指定大表，尽量不存内存：`/*+STREAMTABLE(table_name) */ `

```shell
select /*+STREAMTABLE(b)*/ * from a join b on a.user_id =b.user_id
```

##### 笛卡尔积

在进行outer join时，会先逐条扫描基准表的每一个记录，对于基准表的每个记录，也会扫描整个补充表进行匹配，相当于做了笛卡尔积

解决方法：把补充表放进内存，只会扫描一次，前提是补充表是小表，可以灵活使用left join/right join实现

#### 大大表关联

##### 连接条件为空

大量的连接条件是空或者是 0 ，最后分到一个reduce中

解决方法：

* 把空值的 key 变成一个字符串加上随机数，就可以把倾斜的数据分到不同的 reduce 上，由于空值本来就是无效数据，这样处理也不能影响其他正常key，并不影响最终结果 

```shell
select * from log a join user b on case when ( a.uid = '0 or a.uid is null) then concat('a.uid',rand ()) else a.uid end = b.user_id
```

* 过滤空值

```shell
select * 
from log a 
join users b 
on (a.user_id is not null and a.user_id = b.user_id); 
```

但是这种方法只适用于join，对于left join，需要保留基准表的全表，可以先过滤空值取非空值，再取空值，将两者union all即可

```sql
select * 
from log a 
left join user b 
on a.uid is not null and a.uid=b.user_id
union all
select * 
from log 
where log.uid is null 
```

##### 连接条件是非空

连接条件是大量的非空值也会分到一个reduce

解决方法：只取有用的那部分记录

```sql
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

##### 连接的数据类型不一致

若a表中user_id类型为int，b表中uid既有string类型也有int类型， 默认的 hash 操作会按照 int 类型的 id 进行分配，这样就会导致所有的 string 类型的 id 就被分到同一个 reducer当中

解决方法：把int类型转换成字符串类型

```sql
select * from users a
join logs b
on (a.usr_id = cast(b.uid as string));
```

#### join过程优化

做笛卡儿积时连接条件的判断比较耗时，join用on不用where，避免过程出现大量结果。如果join过程中不加on或者无效on，会使用一个reduce

### count distinct

这两个组合相当于做了分组+排序，若空值或某个key值太多，也会造成数据倾斜

解决方法和join的大大表关联一致，过滤空值或者只取有用值

```shell
select cast(count(distinct(user_id))+1 as bigint ) as user_cnt # +1是因为空值也算作一个uid
from t
where user_id is not null and user_id <>'';
```

### group by

如果没有group by，只有一个reduce；group by会有多个reduce，但相同的key是一个reduce处理，key比较多的reduce会形成数据倾斜

解决方法：在reduce聚合之前，预聚合一次，但预聚合不按key，而是随机分配。这样某个key的数据不会过多，reduce的数据量变小，负载均衡

```shell
hive.groupby.skewindata=true
```

### 其他解决方法

#### map端优化

combiner：在map端做聚合

```shell
hive.map.aggr=true 
```

#### reduce端优化

设置reduce个数

```shell
set mapred.reduce.tasks=10
```

#### 设置并行

```shell
set hive.exec.paralled = true
```

## map优化

控制map数：map数等于切片数，所以控制map个数就是变相控制block size

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

## reduce优化

* 设置reduce任务处理的数据量

```shell
hive.exec.reducers.bytes.per.reducer
```

* 控制reduce个数

**order by：**是全局排序，所以只有一个reduce（多个reduce只能保证局部有序，无法保证全局）。当数据量过大时，一个reduce处理性能不佳

解决方法：将order by用distribute by和sort by结合替换，可以产生多个reduce

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

## 表连接的优化

- 使用相同的连接键：多表连接时，连接键相同，只会产生一个mapreduce job
- 尽早过滤数据：使用尽量少的字段，分区表要分区
- 逻辑复杂时引入中间表：当某个子查询建表经常使用时，可以提取出来

## 其他优化

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

启动一个独立的map-reduce任务来进行文件merge，可以将多个小文件合并

```shell
hive.merge.size.per.task = 256 * 1024 *1024
```
