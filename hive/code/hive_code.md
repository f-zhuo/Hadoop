# 基本命令

创建数据库

```shell
create database myhive comment '测试库';
```

查看数据库信息

```shell
desc database extended myhive;
```

使用数据库

```shell
use myhive;
```

查看数据库

```shell
show databases；
```

删除数据库

hive中删除数据库要求数据库为空（空表也不可以），可以使用cascade强制删除

```shell
drop database if exists myhive cascade;
```

创建表，同时导入数据

 ```shell
CREATE Table movie_table
(movieid STRING,
title STRING,
genres STRING
)
row format delimited fields terminated by ',' 
stored as textfile
location '/movie_table';
# location 后的路径必须是目录
 ```

hive创建表格导入数据时，会读取目录下的所有文件

不在hive终端执行

```shell
hive -f create_movie_table.sql

# cat create_movie_table.sql

CREATE EXTERNAL Table movie_table
(movieid STRING,
title STRING,
genres STRING
)
row format delimited fields terminated by ','
stored as textfile
location '/movie_table';
```

like建表法（只有元数据）

```shell
create table empty_store like store;
```

从本地导入

```shell
load data local inpath '/home//hive_test/movies/movies.csv' overwrite into table movie_table;
```

从HDFS导入

```shell
load data inpath '/movie_table' overwrite into table movie_table;
```

查询建表法（有数据）

```shell
create table t1 as select userid from t2 ;
```

插入数据

```shell
insert into table t1 select usrid, age from t2 limit 3; 
```

导出到本地

```shell
insert overwrite local directory '/home/hive_test/t1' select * from t1;
```

导出到HDFS

```shell
insert overwrite directory '/t1' select * from t1;
```

 join

将A和B进行join

 ```shell
select B.userid, A.title
from movie_table A
join rating_table B
on A.movie_id = B.movie_id
limit 10;
 ```

分区插入

```shell
insert into table t1 partition(c) select * from t2;
```

partition分区

减少查询时扫描的数据，避免全表扫描，提高效率

建分区表

```shell
create table logs
(
ts int,
line string
)
partitioned by(dt string,country string);
load data local inpath '/home/hadoop/file1' into table logs
partition(dt='2001-01-01',country='CH');
load data local inpath '/home/hadoop/file2' into table logs
partition(dt='2001-01-01',country='US');
```

查看表的分区

```shell
show partitions logs;
```

bucket

```shell
set hive.enforce.bucketing=true;

CREATE Table table_b
(userid STRING,
movieid STRING,
rating STRING
)
clustered by (userid) INTO 16 buckets;

insert overwrite table table_b
select userid,movieid,rating from table_a;
```

修改表名

```shell
alter table logs rename to tab_fq;
```

修改列名

```shell
alter table tab_fq change sex gender string;
```

增加列

```shell
alter table tab_fq add columns(age int);
```

删除表

```shell
drop table if exists tab_fq;
```

# transform

自定义函数，创建新的字段

`transform(旧表字段名) using "文件" as 新表字段名`

* awk shell方式

```shell
# cat transform.awk 

{
    print $1"_"$2
}
```

 ```shell
add file /home/hive_test/transform/transform.awk;

select transform(movieId,title) using "awk -f transform.awk" as t from movie_table limit 10;
 ```

* python方式

```shell
# cat transform.py

import sys

for line in sys.stdin:
     ss = line.strip().split('\t')
     print('_'.join([ss[0].strip(), ss[1].strip()]))
```

```shell
add file /home/hive_test/transform/transform.py;

select transform(movieId,title) using "python transform.py" as t from movie_table limit 10;
```

## 利用hive实现wordcount

 ```shell
select word, count(*) form t group by word;
 ```

```shell
# cat mapper.py

import sys

for line in sys.stdin:
     ss = line.strip().split(' ')
     for word in ss:
        print('%s\t1' % (word))


# cat reducer.py

import sys
 
last_key = None
last_count = 0

for line in sys.stdin:
     ss = line.strip().split('\t')
     if len(ss) != 2:
         continue
     key = ss[0].strip()
     count = ss[1].strip()

     if last_key and last_key != key:
         print('%s\t%d' % (last_key, last_count))
         last_key = key
         last_count = int(count)
     else:
         last_key = key
         last_count += int(count)

print('%s\t%d' % (last_key, last_count))

 

create table docs(line string);

load data local inpath '/home/badou/hive_test/The_Man_of_Property.txt' into table docs;


create table word_count(word string ,count string);

insert overwrite table word_count
select transform(wc.word, wc.count) using 'python reducer.py' as word,count
from
(select transform(line) using 'python mapper.py' as word,count from docs cluster by word) wc;
```







 

