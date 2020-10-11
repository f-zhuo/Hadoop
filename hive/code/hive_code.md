# 基本命令

## 数据库

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
show databases;
```

删除数据库

hive中删除数据库要求数据库为空（空表也不可以），可以使用cascade强制删除

```shell
drop database if exists myhive cascade;
```

## 表和导入导出

### 内部表（默认）

* 建表的同时导入数据

 ```shell
CREATE Table movie_table
(movieid STRING,
title STRING,
genres STRING
)
row format delimited fields terminated by ',' 
stored as textfile
location '/movie_table';
# location后的路径必须是目录,hive导入数据时，会读取目录下的所有文件
 ```

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

* like建表法（只有元数据）

```shell
create table empty_store like store;
```

从本地导入

```shell
load data local inpath '/home/hive_test/movies/movies.csv' overwrite into table movie_table;
```

从HDFS导入

```shell
load data inpath '/movie_table' overwrite into table movie_table;
```

* 查询建表法（有数据）

```shell
from t2 create table t1 as select userid;
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

### 外部表

```sql
create external table t1 like t2;
```

### partition

```shell
create table logs
(ts int,
line string)
partitioned by(dt string,country string);

load data local inpath '/home/hadoop/file1' into table logs
partition(dt='2001-01-01',country='CH');
load data local inpath '/home/hadoop/file2' into table logs
partition(dt='2001-01-01',country='US');
```

分区插入

```shell
insert into table t1 partition(c) select * from t2;
```

查看logs表的分区

```shell
show partitions logs;
```

### bucket

```shell
set hive.enforce.bucketing=true;

CREATE Table table_b
(userid STRING,
movieid STRING,
rating STRING
)
clustered by (userid) INTO 16 buckets;

insert overwrite table t2
select userid,movieid,rating from t1;
```

## 基本查询语句

- SELECT  A  FROM  B  WHERE  C

选出城市在北京，性别为女的10个用户名

```sql
SELECT user_name
FROM user_info
WHERE city='beijing' 
      and sex='female'
limit 10;
```

**如果查询的表为一个分区表，则WHERE条件必须对分区字段进行限制**

选出在2019年4月9日，购买商品品类为food的用户名，购买数量，支付金额（user_trade的分区字段是dt）

```sql
SELECT user_name,
       piece,
       pay_amount
FROM user_trade
WHERE goods_category='food' and dt='2019-04-09';
```

选出购买商品品类为food的用户名，购买数量，支付金额（user_trade的分区字段是dt）

*对分区字段没有限制条件可令其大于0*

```sql
SELECT user_name,
       piece,
       pay_amount
FROM user_trade
WHERE goods_category='food' and dt>'0';
```

- GROUP BY / GROUP BY  HAVING

2019年1月至4月，每个品类有多少人购买，累计金额是多少

```sql
SELECT goods_category,
       count(distinct user_name) as user_num,
       sum(pay_amount) as total_amount
FROM user_trade
WHERE dt between '2019-01-01' and '2019-04-30'
GROUP BY goods_category;
```

HAVING和WHERE的区别：

WHERE是限制原始表数据，HAVING与GROUP BY连用，限制分组后的数据

2019年4月，支付金额超过5万元的用户

```sql
SELECT user_name,
       sum(pay_amount) as total_amount
FROM user_trade
WHERE dt between '2019-04-01' and '2019-04-30'
GROUP BY user_name HAVING sum(pay_amount)>50000;
```

- ORDER BY

2019年4月，支付金额最多的TOP5用户

```sql
SELECT user_name,
       sum(pay_amount) as total_amount
FROM user_trade
WHERE dt between '2019-04-01' and '2019-04-30'
GROUP BY user_name
ORDER BY total_amount DESC
limit 5;
```

**执行顺序**

FROM - ON - JOIN - WHERE - GROUP BY - HAVING - SELECT - DISTINCT - ORDER BY - LIMIT

## 常用函数

### 日期函数

#### 时间戳转换为日期

- `from_unixtime(timestamp,'yyyy-MM-dd hh:mm:ss')`

```sql
SELECT pay_time,
       from_unixtime(pay_time,'yyyy-MM-dd hh:mm:ss') 
FROM user_trade
WHERE dt='2019-04-09';
```

#### 日期转换为时间戳

- `unix_timestamp(string date)`

```sql
SELECT pay_time,
       from_unixtime(pay_time,'yyyy-MM-dd hh:mm:ss') unix,
       unix_timestamp(from_unixtime(pay_time,'yyyy-MM-dd hh:mm:ss'))
FROM user_trade
WHERE dt='2019-04-09';
```

错误写法

```sql
SELECT pay_time,
       from_unixtime(pay_time,'yyyy-MM-dd hh:mm:ss') unix,
       unix_timestamp(unix)
```

#### 日期格式转换

* `to_date(time)`： 把datetime（包含年月日小时分钟秒）转换为date（仅含年月日）的格式

```sql
SELECT to_data(pay_time)
FROM user_trade
WHERE dt='2019-04-09';       
```

#### 计算日期间隔

- `datediff(end_date,start_date)`

```sql
SELECT user_name,
       datediff('2019-05-01',to_date(firstactivetime))
FROM user_info
limit 10;
```

#### 日期增加/减少函数

- `date_add(string startdate,int adding_days)`
- `date_sub(string startdate,int sub_days)`

```sql
SELECT user_name,
       dt,
       date_add(dt,7),
       date_sub(dt,7)
FROM user_trade
limit 10;
```

### 条件函数

#### case when（多条件）

统计20岁以下，20-30岁，30-40岁，40岁以上的用户数

```sql
SELECT case when age <20 then '20以下'
            when 20<=age and age<30 then '20-30'
            when 30<=age and age<40 then '30-40'
            else '40以上' end,
       count(distinct user_name) user_num
FROM user_info
GROUP BY case when age <20 then '20以下'
              when 20<=age and age<30 then '20-30'
              when 30<=age and age<40 then '30-40'
              else '40以上' end;
```

错误写法：连续不等条件必须分开写

```sql
SELECT case when age <20 then '20以下'
            when 20<=age<30 then '20-30'
            when 30<=age<40 then '30-40'
            else '40以上' end
```

错误写法：group by不可重命名

```sql
GROUP BY (case when age <20 then '20以下'
               when 20<=age and age<30 then '20-30'
               when 30<=age and age<40 then '30-40'
               else '40以上' end) age;
```

#### if（两种情况）

统计每个性别用户等级高低（level>5即为高级）的分布情况

```sql
SELECT sex,
       if(level>5,'高级','低级'),
       count(distinct user_name)
FROM user_info
GROUP BY sex,
         if(level>5,'高级','低级');
```

### 字符串函数

#### 截取字符串

- `substr(string,start,len)`：若不指定截取长度，则从起始位置截取到最后

每个月新激活的用户数

```sql
SELECT substr(firstactivetime,1,7),
       count(distinct user_name)
FROM user_info
GROUP BY substr(firstactivetime,1,7);
```

#### json和map格式的取值函数

- json格式：`get_json_object(json_string,'$.key')`
- map格式：`map_field['key']`

不同手机品牌的用户数

json格式

```sql
SELECT get_json_object(extra1,'$.phonebrand'),
        count(distinct user_id)
FROM user_info
GROUP BY get_json_object(extra1,'$.phonebrand');
```

map格式

```sql
SELECT extra2['phonebrand'],
       count(distinct user_id)
FROM user_info
GROUP BY extra2['phonebrand'];
```

错误写法：distinct和group by不可同用在一个查询里，group by相当于distinct

```sql
SELECT distinct extra2['phonebrand'],
       count(distinct user_id)
FROM user_info
GROUP BY extra2['phonebrand'];
```

### 聚合统计函数

|       函数       |   含义   |
| :--------------: | :------: |
|     count()      |   计数   |
| count(distinct ) | 去重计数 |
|      sum()       |   求和   |
|      avg()       |  平均值  |
|      max()       |  最大值  |
|      min()       |  最小值  |

ELLA用户的2018年的平均支付金额，2018年最大的支付日期与最小支付日期的间隔

```sql
SELECT avg(pay_amount),
       datediff(from_unixtime(max(pay_time),'yyyy-MM-dd'),from_unixtime(min(pay_time),'yyyy-MM-dd'))
FROM user_trade
WHERE user_name='ELLA' and 
      substr(dt,1,4)='2018';
```

或

```sql
SELECT avg(pay_amount),
       datediff(max(from_unixtime(pay_time,'yyyy-MM-dd')),min(from_unixtime(pay_time,'yyyy-MM-dd')))
FROM user_trade
WHERE user_name='ELLA' and 
      year(dt)='2018';
```

*统计聚合函数不可做如avg(count(* ))这样的嵌套组合

2018年购买的商品品类在两个以上的用户数

```sql
SELECT count(a.user_name)
FROM
 (SELECT user_name,
         count(distinct goods_category) as category_num
 FROM user_trade
 WHERE year(dt)='2018'
 GROUP BY user_name HAVING count(distinct goods_category)>2)a; 
```

*一旦分组，select后面跟的字段必须是在group by后的，但聚合函数除外*

用户激活时间在2018年，年龄在20-30和30-40的婚姻状况分步

```sql
SELECT a.age,
        if(a.marrige='1','已婚','未婚'),
        count(distinct a.user_id)
 FROM
     (SELECT user_id,
             (case when age<20 then '20以下'
                   when 20<=age and age<30 then '20-30'
                   when 30<=age and age<40 then '30-40'
                   else '40以上' end) age,
                   get_json_object(extra1, '$.marriage_status') marrige
     FROM user_info
     WHERE substr(firstactivetime,1,4) ='2018') a
 WHERE a.age in ('20-30','30-40')
 GROUP BY a.age,
          if(a.marrige='1','已婚','未婚');
```

激活天数距今超过300天的男女分布情况(user_info)

```sql
SELECT sex,
        count(distinct user_id)
 FROM user_info
 WHERE datediff('2019-08-07',to_date(firstactivetime))>'300'
 GROUP BY sex;
```

不同性别，教育程度的分布情况(user_info)

```sql
SELECT sex,
        extra2['education'],
        count(distinct user_id)
 FROM user_info
 GROUP BY sex,
          extra2['education'];
```

2019年1月1日到2019年4月30日，每个时段的不同品类购买金额分布(user_trade)

```sql
SELECT month(dt),
        goods_category,
        case when pay_amount<100 then '100以下'
                   when 100<=pay_amount and pay_amount<500 then '100-500'
                   when 500<=pay_amount and pay_amount<1000 then '500-1000'
                   else '1000以上' end,
        count(distinct user_id)
 FROM user_trade
 WHERE dt between '2019-01-01' and '2019-04-30'
 GROUP BY month(dt),
          goods_category,
          case when pay_amount<100 then '100以下'
                   when 100<=pay_amount and pay_amount<500 then '100-500'
                   when 500<=pay_amount and pay_amount<1000 then '500-1000'
                   else '1000以上' end;
```

### coalesce函数

 ` coalesce(expression1,expression2,expression3,……)`

依次访问expression1，expression2，expression3……遇到非null值就返回，遇到null值就访问下一个。若所有的都为null值，返回null值

# 表连接

## inner join/join

内连接，返回满足连接条件的两个表的所有记录。内连接必须重命名，inner可以省略，必须要用on作唯一条件连接，不能用where，否则会先进行笛卡尔积再过滤

找出既在user_list_1,又在user_list_2的用户

```sql
SELECT * 
FROM user_list_1 a 
INNER JOIN user_list_2 b on a.user_id=b.user_id;

SELECT * 
FROM user_list_1 a 
JOIN user_list_2 b on a.user_id=b.user_id;
```

在2019年购买后又退款的用户

```sql
SELECT a.user_name
FROM
    (SELECT distinct user_name
    FROM user_trade 
    WHERE year(dt)='2019') a 
    JOIN 
    (SELECT distinct user_name
    FROM user_refund 
    WHERE year(dt)='2019') b 
     on a.user_name=b.user_name;
```

在2017年和2018年都购买的用户

```sql
SELECT a.user_name
 FROM
     (SELECT distinct user_name
     FROM user_trade
     WHERE year(dt)='2017') a
     JOIN
     (SELECT distinct user_name
     FROM user_trade
     WHERE year(dt)='2018') b
     on a.user_name=b.user_name;
```

在2017年，2018年，2019年都有交易的用户

```sql
SELECT a.user_name
 FROM 
     (SELECT distinct user_name
     FROM trade_2017) a 
     JOIN
     (SELECT distinct user_name
     FROM trade_2018) b
     on a.user_name=b.user_name
     JOIN
     (SELECT distinct user_name
     FROM trade_2019) c
     on b.user_name=c.user_name;
```

## left join

左连接，属于外连接（outer join）。以左表为基准，返回和左表相同的记录数。从左表第一行开始，根据on连接条件循环扫描右表所有记录，存在匹配结果就连接，否则用NULL连接

user_list_1,user_list_2的左连接

```sql
SELECT *
 FROM user_list_1 a
 LEFT JOIN user_list_2 b on a.user_id=b.user_id; 
```

## right join

右连接，属于外连接。以右表为基准，返回和右表相同的记录数。从右表第一行开始，根据on连接条件循环扫描左表所有记录，若存在匹配结果就连接，否则用NULL连接

取出在user_list_1表但不在user_list_2表的用户

*null值一定要用 is,用 =会出错*

```sql
SELECT a.user_name
FROM
    (SELECT distinct user_name
    FROM user_list_1) a
    LEFT JOIN
    (SELECT distinct user_name
    FROM user_list_2) b
    ON a.user_name=b.user_name
WHERE b.user_name is null;
```

取出既在user_list_1表也在user_list_2表的用户

```sql
SELECT b.user_name
FROM
    (SELECT distinct user_name
    FROM user_list_1) a
    LEFT JOIN
    (SELECT distinct user_name
    FROM user_list_2) b
    ON a.user_name=b.user_name
WHERE b.user_name is not null;

SELECT b.user_name
FROM
    (SELECT distinct user_name
    FROM user_list_1) a
    JOIN
    (SELECT distinct user_name
    FROM user_list_2) b
    ON a.user_name=b.user_name;
```

在2019年购买，但是没有退款的用户

```sql
SELECT a.user_name
FROM
    (SELECT distinct user_name
    FROM user_trade
    WHERE year(dt)='2019') a
    LEFT JOIN
    (SELECT distinct user_name
    FROM user_refund
    WHERE year(dt)='2019') b
    ON a.user_name=b.user_name
WHERE b.user_name is null;
```

2019年购买用户的学历分布

```sql
SELECT b.education,
       count(a.user_name)
FROM
    (SELECT distinct user_name
    FROM user_trade
    WHERE year(dt)='2019') a
    LEFT JOIN
    (SELECT distinct user_name,
            get_json_object(extra1,'$.education') education
    FROM user_info) b
    on a.user_name=b.user_name
GROUP BY b.education;
```

在2017年和2018年都购买，但2019年没有购买的用户

```sql
SELECT a.user_name
FROM
    (SELECT distinct user_name
    FROM trade_2017) a
    JOIN
    (SELECT distinct user_name
    FROM trade_2018) b
    ON a.user_name=b.user_name
    LEFT JOIN
    (SELECT distinct user_name
    FROM trade_2019) c
    ON b.user_name=c.user_name
 WHERE c.user_name is null;
```

## full join

全连接，属于外连接。返回左表和右表去重后的记录数之和，两表互相匹配，存在匹配结果就连接，否则用NULL连接

*对于outer join，on连接条件是不可缺少的*

对两个表进行full join

```sql
SELECT *
FROM user_list_1 a 
FULL JOIN user_list_2 b ON a.user_id=b.user_id;
```

user_list_1和user_list_2的所有用户

```sql
SELECT coalesce(a.user_name,b.user_name)
FROM user_list_1 a FULL JOIN 
user_list_2 b ON a.user_id=b.user_id; 
```

## left semi join

返回左表中的on字段也在右表中的记录，记录数不一定等于左表

选出user_list_2中包含user_list_1的user_id和user_name

```sql
SELECT a.user_id,
	   a.user_name
FROM user_list_1 a LEFT SEMI JOIN 
user_list_2 b ON a.user_id=b.user_id;
```

等价于

```sql
SELECT user_id,
	   user_name
FROM user_list_1 
WHERE user_id in 
(select user_id from user_list_2);
```

## cross join

笛卡尔连接。对两个表做笛卡尔积，所以不需要也不允许指定连接条件

```mysql
SELECT *
FROM user_list_1 a
CROSS JOIN user_list_2 b;
```

当inner join没有指定on连接时，效果和cross join是一样的。上面的cross join就可以写成

```mysql
SELECT *
FROM user_list_1 a
JOIN user_list_2 b;
```

## union all

合并，把两个/多个表合并为一个表，类似追加记录。要求合并的表的字段名，字段顺序一致，否则会出错。union all只是把表合并，故不需要连接条件

把user_list_1和user_list_2合并为一个表

```sql
SELECT user_id,
        user_name
 FROM user_list_1 
 UNION ALL
 SELECT user_id,
        user_name
 FROM user_list_2; 
```

2017年到2019年有交易的所有用户数

```sql
 SELECT count(distinct a.user_name)
 FROM
     (SELECT distinct user_name
     FROM trade_2017
     UNION ALL
     SELECT distinct user_name
     FROM trade_2018
     UNION ALL
     SELECT distinct user_name
     FROM trade_2019) a;
```

## union

合并，用法同union all，区别在于union会去重，会排序；union all不会去重，不会排序。故union all效率更高，尽量使用union all

2019年每个用户的支付和退款金额汇总

```sql
SELECT coalesce(a.user_name,b.user_name),
        a.pay_total,
        b.refund_total
 FROM
     (SELECT user_name,
     sum(pay_amount) pay_total
     FROM user_trade
     WHERE year(dt)='2019'
     GROUP BY user_name) a
     FULL JOIN
     (SELECT user_name,
     sum(refund_amount) refund_total
     FROM user_refund
     WHERE year(dt)='2019'
     GROUP BY user_name) b
     ON a.user_name=b.user_name;
     

SELECT a.user_name,
        sum(a.pay_total),
        sum(a.refund_total)
 FROM
     (SELECT user_name,
             sum(pay_amount) pay_total,
             0 as refund_total
     FROM user_trade
     WHERE year(dt)='2019'
     GROUP BY user_name 
     UNION ALL
     SELECT user_name,
            0 as pay_total,
            sum(refund_amount) refund_total
     FROM user_refund
     WHERE year(dt)='2019'
     GROUP BY user_name) a
GROUP BY a.user_name;
```

2019年每个支付用户的支付金额和退款金额

```sql
SELECT a.user_name,
       a.pay_total,
       b.refund_amount
FROM
    (SELECT user_name,
           sum(pay_amount) pay_total
    FROM user_trade
    WHERE year(dt)='2019'
    GROUP BY user_name) a
    LEFT JOIN
    (SELECT user_name,
           sum(refund_amount) refund_amount
    FROM user_refund
    WHERE year(dt)='2019'
    GROUP BY user_name) b
    ON a.user_name=b.user_name;
```

首次激活时间在2017年，但是一直没有支付的用户年龄段分布

```sql
SELECT a.age,
        count(a.user_name)
 FROM
     (SELECT user_name,
             case when age<20 then '20以下'
                   when 20<=age and age<30 then '20-30'
                   when 30<=age and age<40 then '30-40'
                   else '40以上' end age
     FROM user_info
     WHERE year(firstactivetime)='2017') a
      LEFT JOIN
      (SELECT distinct user_name
      FROM user_trade
      WHERE dt>0) b
      ON a.user_name=b.user_name
 WHERE b.user_name is null
 GROUP BY a.age;
```

2018,2019年交易的用户，其激活时间分布

```sql
SELECT c.activetime,
       count(c.user_name)
FROM
    (SELECT distinct a.user_name
    FROM
        (SELECT distinct user_name
        FROM trade_2018
        UNION ALL
        SELECT distinct user_name
        FROM trade_2019) a) b
     LEFT JOIN
     (SELECT user_name,
             hour(firstactivetime) activetime
     FROM user_info) c
     ON b.user_name=c.user_name
GROUP BY c.activetime;


SELECT hour(firstactivetime),
       count(a.user_name)
FROM
    (SELECT user_name
    FROM trade_2018
    UNION
    SELECT user_name
    FROM trade_2019)a
    LEFT JOIN user_info b on a.user_name=b.user_name
GROUP BY hour(firstactivetime);
```

在2019年购买后又退款的用户性别分布

```sql
SELECT sex,
       count(d.user_name)
FROM
    (SELECT b.user_name
    FROM
        (SELECT distinct user_name
        FROM user_trade
        WHERE year(dt)='2019') a
        JOIN
        (SELECT distinct user_name
        FROM user_refund
        WHERE year(dt)='2019') b
        ON a.user_name=b.user_name) c
     LEFT JOIN
     (SELECT distinct user_name,
             sex
     FROM user_info) d
     ON c.user_name=d.user_name
 GROUP BY sex;
```

在2018年购买，但是没在2019年购买的用户城市分布

```sql
SELECT city,
       count(d.user_name)
FROM
    (SELECT a.user_name
    FROM
        (SELECT distinct user_name
        FROM trade_2018) a
        LEFT JOIN
        (SELECT distinct user_name
        FROM trade_2019) b
        ON a.user_name=b.user_name
    WHERE b.user_name is null
        ) c
     LEFT JOIN
     (SELECT distinct user_name,
             city
     FROM user_info) d
     ON c.user_name=d.user_name
 GROUP BY city;
```

2017-2019年，有交易但是没有退款的用户的手机品牌分布

```sql
SELECT e.phonebrand,
       count(e.user_name)
FROM
    (SELECT b.user_name
    FROM
        (SELECT distinct a.user_name
        FROM
            (SELECT distinct user_name
            FROM trade_2017
            UNION ALL
            SELECT distinct user_name
            FROM trade_2018
            UNION ALL
            SELECT distinct user_name
            FROM trade_2019) a ) b
        LEFT JOIN
        (SELECT user_name
        FROM user_refund) c
        ON b.user_name=c.user_name
     WHERE c.user_name is null) d
     LEFT JOIN
     (SELECT distinct user_name,
             extra2['phonebrand'] phonebrand
     FROM user_info) e
     ON d.user_name=e.user_name
 GROUP BY e.phonebrand;
```

其实union和union all并不属于表连接

# 窗口函数

## 累计计算窗口函数

### 求和计算函数

```sql
sum() over(partition by  order by  rows between  and  )
```

partition是分组，order是排序

有x1，x2，x3，x4，x5……的值，求某数前面所有值的和，如x1,x1+x2,x1+x2+x3……

2018年每月的支付总额和当年累计支付总额

```sql
SELECT a.month,
       a.pay_total,
       sum(a.pay_total) over(order by a.month)
FROM
    (SELECT month(dt) month,
            sum(pay_amount) pay_total
     FROM user_trade
     WHERE year(dt)=2018
     GROUP BY month(dt)) a;
```

2017-2018年每月的支付总额和当年累积支付总额

```sql
SELECT a.year,
       a.month,
       a.pay_total,
       sum(a.pay_total) over(partition by a.year order by a.month)
FROM
    (SELECT year(dt) year,
            month(dt) month,
            sum(pay_amount) pay_total
    FROM user_trade
    WHERE year(dt) in (2017,2018)
    GROUP BY year(dt),
             month(dt)) a;
```

计算每12个月的用户累计支付金额

```sql
SELECT a.user_name,
       a.year,
       a.month,
       a.pay_total,
       sum(a.pay_total) over(partition by a.year,a.user_name order by a.month)
FROM
    (SELECT user_name,
            year(dt) year,
            month(dt) month,
            sum(pay_amount) pay_total
    FROM user_trade
    WHERE dt>'0'
    GROUP BY user_name,
             year(dt),
             month(dt)) a;
```

2017、2018年按月累计去重的购买用户数

```sql
SELECT b.year,
        b.month,
        sum(b.user_num) over(partition by b.year order by b.month)
 FROM
     (SELECT a.year,
            a.month,
            count(distinct user_name) user_num
     FROM
         (SELECT year(dt) year,
                 min(month(dt)) month
         FROM user_trade
         WHERE year(dt) in (2017,2018)
         GROUP BY year(dt) year,
                  user_name) a
     GROUP BY a.year,
              a.month) b


 set hive.mapred.mode=nonstrict;
 SELECT b.month,
 count(distinct a.user_name)
 FROM
 (SELECT substr(dt,1,7) as month,
 user_name
 FROM user_trade
 WHERE year(dt) in (2017,2018)
 GROUP BY substr(dt,1,7),
 user_name)a
 CROSS JOIN
 (SELECT month
 FROM dim_month)b
 WHERE b.month>=a.month
 and substr(a.month,1,4)=substr(b.month,1,4)
 GROUP BY b.month;
```

2017、2018年按月累计去重的退款用户数

```sql
SELECT b.year,
         b.month,
         sum(b.user_num) over(partition by b.year order by b.month)
  FROM
      (SELECT a.year,
             a.month,
             count(distinct user_name) user_num
      FROM
          (SELECT year(dt) year,
                 min(month(dt)) month
          FROM user_refund
          WHERE year(dt) in (2017,2018)
          GROUP BY year(dt),
                   user_name) a
      GROUP BY a.year,
               a.month) b;
```

### 移动平均函数

```sql
avg() over(partition by  order by  rows between  and )
```

计算固定个数的移动数值的平均值，如有x1，x2，x3，x4，x5……的值，(x1+x2)/2,(x2+x3)/2,(x3+x4)/2,……

`rows between 2 preceding and current row`：前两行和本行

`rows between 3 preceding and 1 following`：前三行到后一行（5行）

`rows between unbounded preceding and current row`：前面所有行和本行

`rows between current row and unbounded following`：本行和后面所有行

2018年每个月的近三月移动平均支付金额

```sql
SELECT a.month,
       a.pay_total,
       avg(a.pay_total) over(order by a.month rows between 2 preceding and current row)
FROM
    (SELECT month(dt) month,
            sum(pay_amount) pay_total
    FROM user_trade
    WHERE year(dt)=2018
    GROUP BY month(dt)) a;
```

对于前两个月，不够三个月计算平均值，第一个月即自身，第二个月是前两个月的平均值

### 累计最值函数

```sql
max() over(partition by  order by  rows between   and )
min() over(partition by  order by  rows between   and )                   
```

2018年每个月的近三月的最大和最小支付金额

```sql
SELECT a.month,
       a.pay_total,
       max(a.pay_total) over(order by a.month rows between 2 preceding and current row),
       min(a.pay_total) over(order by a.month rows between 2 preceding and current row) 
FROM
    (SELECT month(dt) month,
            sum(pay_amount) pay_total
    FROM user_trade
    WHERE year(dt)=2018
    GROUP BY month(dt)) a;
```

计算每4个月的最大退款金额

```sql
SELECT substr(dt,1,7),
       sum(refund_amount),
       max(sum(refund_amount)) over(order by substr(dt,1,7) rows between 3 preceding and current row)
FROM user_refund
WHERE dt>'0'
GROUP BY substr(dt,1,7);
```

## 分区排序窗口函数

为每行记录生成一个序号，从1开始，依次排序，不重复

```sql
row_number() over(partition by  order by )
```

按字段值排序，并列时排名相同，下一个不同值的排名从并列后的真实位置开始，排序是不连续的

```sql
rank() over(partition by  order by )
```

按字段值排序，并列时排名相同，下一个不同值的排名从并列排名开始，排序是连续的

```sql
dense_rank() over(partition by  order by )
```

2019年1月，用户购买商品品类数量的排名

```sql
SELECT user_name,
       count(distinct goods_category),
       row_number() over(order by count(distinct goods_category)),
       rank() over(order by count(distinct goods_category)),
       dense_rank() over(order by count(distinct goods_category))
FROM user_trade
WHERE substr(dt,1,7)='2019-01'
GROUP BY user_name;
```

选出2019年支付金额排名在第10，20，30名的用户

```sql
SELECT a.user_name,
       a.pay_total,
       a.level
FROM
    (SELECT user_name,
            sum(pay_amount) pay_total,
            (rank() over(order by sum(pay_amount) desc)) level 
    FROM user_trade
    WHERE year(dt)=2019
    GROUP BY user_name) a
WHERE a.level in (10,20,30);
```

每个城市，不同性别，2018年支付金额最高的top3用户

```sql
SELECT c.user_name,
       c.city,
       c.sex,
       c.pay_total,
       c.rank
FROM
    (SELECT a.user_name,
            b.city,
            b.sex,
            a.pay_total,
            row_number() over(partition by b.city, b.sex order by a.pay_total desc) rank
    FROM
        (SELECT user_name,
                sum(pay_amount) pay_total
        FROM user_trade
        WHERE year(dt)='2018'
        GROUP BY user_name) a
        LEFT JOIN
        (SELECT user_name,
                city,
                sex
        FROM user_info) b
        ON a.user_name=b.user_name) c
WHERE c.rank<=3;
```

## 分组排序窗口函数

```sql
ntile(n) over(partition by  order by )
```

ntile不支持rows between and，是对整个数据做分组。若分组不均匀，则默认增加第一个分组的分布，分组结果为1，2，……n

2019年1月的支付用户，按照支付金额分成5组

```sql
SELECT user_name,
       sum(pay_amount) pay_total,
       ntile(5) over(order by sum(pay_amount) desc)
FROM user_trade
WHERE substr(dt,1,7)='2019-01'
GROUP BY user_name;
```

选出2019年退款金额排名前10%的用户

```sql
SELECT a.user_name,
       a.refund_total,
       a.level
FROM
    (SELECT user_name,
            sum(refund_amount) refund_total,
            ntile(10) over(order by sum(refund_amount) desc) level
    FROM user_refund
    WHERE year(dt)=2019
    GROUP BY user_name) a
 WHERE a.level=1;
```

每个手机品牌退款金额前25%的用户

```sql
SELECT c.user_name,
       c.refund_total,
       c.phonebrand,
       c.rank
FROM
    (SELECT a.user_name,
           a.refund_total,
           b.phonebrand,
           ntile(4) over(partition by b.phonebrand order by a.refund_total desc) rank
    FROM
        (SELECT user_name,
                sum(refund_amount) refund_total
        FROM user_refund
        WHERE dt>'0'
        GROUP BY user_name) a
        LEFT JOIN
        (SELECT user_name,
                extra2['phonebrand'] phonebrand
         FROM user_info) b
         ON a.user_name=b.user_name) c 
WHERE c.rank=1;
```

## 偏移分析窗口函数

往前偏移

```sql
lag(field,offset,default) over(partition by……order by……)
```

往后偏移

```sql
lead(field,offset,default) over(partition by……order by……)
```

field是偏移的字段名，offset是偏移量，default是偏移量超出字段范围时的默认值，若不指定，则返回null

Alice和Alexander的时间偏移

```sql
SELECT user_name,
       dt,
       lag(dt) over(partition by user_name order by dt),
       lag(dt,2,dt) over(partition by user_name order by dt),
       lead(dt) over(partition by user_name order by dt),
       lead(dt,2,dt) over(partition by user_name order by dt)
FROM user_trade
WHERE user_name in ('Alice','Alexander')
      and dt>'0';
```

支付时间间隔超过100天的用户数

```sql
SELECT count(distinct user_name)
FROM
    (SELECT distinct user_name,
            dt,
            lead(dt) over(partition by user_name order by dt) lead_dt
    FROM user_trade
    WHERE dt>'0') a
WHERE a.lead_dt is not null
      and datediff(a.lead_dt,a.dt)>100;
```

退款时间间隔最长的用户

```sql
SELECT a.user,
       max(datediff(a.lead_dt,a.dt))
FROM
    (SELECT user_name,
            dt,
           lead(dt) over(partition by user_name order by dt) lead_dt
    FROM user_refund
    WHERE lead(dt) over(partition by user_name order by dt) is not null
          and dt>'0'
    GROUP BY user_name) a;  
```

# HiveSQL常用技巧

## 去重

用group by替换distinct

取出user_trade表中全部支付用户

* distinct写法

```sql
SELECT distinct user_name
 FROM user_trade
 WHERE dt>'0';
```

* group by优化写法

```sql
SELECT user_name
 FROM user_trade
 WHERE dt>'0'
 GROUP BY user_name;
```

在2019年购买后又退款的用户

```sql
SELECT a.user_name
 FROM
     (SELECT distinct user_name
     FROM user_trade
     WHERE year(dt)='2019') a
     JOIN
     (SELECT distinct user_name
     FROM user_refund
     WHERE year(dt)='2019')b ON a.user_name=b.user_name;
```

优化写法

```sql
SELECT a.user_name
 FROM
     (SELECT user_name
     FROM user_trade
     WHERE year(dt)='2019'
     GROUP BY user_name) a
     JOIN
     (SELECT user_name
     FROM user_refund
     WHERE year(dt)='2019'
     GROUP BY user_name)b ON a.user_name=b.user_name;
```

## 排序组合

### grouping sets

形式是个集合，相当于对集合里的所有分组进行group by，然后union all

用户的性别分布，城市分布，等级分布

```sql
SELECT sex,
       count(user_name)
FROM user_info
GROUP BY sex,
         user_name;
           
SELECT city,
       count(user_name)
FROM user_info
GROUP BY city,
         user_name; 
           
SELECT level,
       count(user_name)
FROM user_info
GROUP BY level,
         user_name;
```

优化

```sql
SELECT sex,
       city,
       level,
       count(user_name)
FROM user_info
GROUP by sex,
         city,
         level,
         user_name
GROUPING SETS((sex,user_name),(city,user_name),(level,user_name));
```

用户的性别分布和每个性别的城市分布

```sql
SELECT sex,
       count(distinct user_id)
FROM user_info
GROUP BY sex;
 
SELECT sex,
       city,
       count(distinct user_id)
FROM user_info
GROUP BY sex,
         city;
```

优化

```sql
SELECT sex,
       city,
       count(distinct user_name)
FROM user_info
GROUP BY sex,city
GROUPING SETS(sex,(sex,city));
```

每个性别、不同性别和手机品牌的退款金额分布

```sql
SELECT c.sex,
        c.phonebrand,
        sum(c.refund_total)
 FROM
     ((SELECT user_name,
             sex,
             extra2['phonebrand']) phonebrand
       FROM user_info) a
       LEFT JOIN
       (SELECT user_name,
               sum(refund_amount) refund_total
        FROM user_refund
        GROUP BY user_name) b
        ON a.user_name=b.user_name) c
  GROUP BY c.sex,
           c.phonebrand
  GROUPING SETS(c.sex,(c.sex,c.phonebrand))
```

### with cube

对group by的所有维度进行组合聚合

性别，城市，等级的各种组合的用户分布

```sql
SELECT sex,
       city,
       level,
       count(distinct user_id)
FROM user_info
GROUP BY sex,city,level
GROUPING SETS(sex,city,level,(sex,city),(sex,level),(city,level),(sex,city,level));
```

优化

```sql
SELECT sex,
       city,
       level,
       count(distinct user_id)
FROM user_info
GROUP BY sex,city,level
with cube;
```

### with rollup

是with cube的子集，以group by**最左侧维度**为基准做层级聚合，主要用于时间上

求每个月的支付总额和每年的支付总额

```sql
SELECT a.dt,
       sum(a.year_total),
       sum(a.month_total),         
FROM 
      (SELECT year(dt) dt,
              sum(pay_amount) year_total,
              0 as month_total
      FROM user_trade
      WHERE dt>'0'
      GROUP BY year(dt)
      UNION ALL
      SELECT substr(dt,1,7) dt,
             0 as year_total,
             sum(pay_amount) month_total
      FROM user_trade
      WHERE dt>'0'
      GROUP BY substr(dt,1,7)) a
GROUP BY a.dt;
```

优化

```sql
SELECT year(dt),
       month(dt),
       sum(pay_amount)
FROM user_trade
WHERE dt>'0'
GROUP BY year(dt),
          month(dt)
with rollup;
```

## 转换思路

在2017年和2018年都购买的用户

```sql
SELECT a.user_name
 FROM
     (SELECT user_name
     FROM user_trade
     WHERE year(dt)='2017'
     GROUP BY user_name) a
     JOIN
     (SELECT user_name
     FROM user_trade
     WHERE year(dt)='2018'
     GROUP BY user_name) b
     ON a.user_name=b.user_name;
```

优化

```sql
SELECT a.user_name
  FROM
      (SELECT user_name,
             count(distinct year(dt)) year_num
      FROM user_trade
      WHERE year(dt) in (2017,2018)
      GROUP BY user_name) a
 WHERE a.year_num=2;
```

## 并发执行

```sql
set hive.exec.parallel=true
```

每个用户的支付和退款金额汇总

```sql
SELECT a.user_name,
        sum(a.pay_amount),
        sum(a.refund_amount)
 FROM
     (
     SELECT user_name,
            sum(pay_amount) as pay_amount,
            0 as refund_amount
     FROM user_trade
     WHERE dt>'0'
     GROUP BY user_name
     UNION ALL
     SELECT user_name,
            0 as pay_amount,
            sum(refund_amount) as refund_amount
     FROM user_refund
     WHERE dt>'0'
     GROUP BY user_name
     )a
 GROUP BY a.user_name;
```

## 行列互转与lateral view

### 行转列

```sql
explode(split(field,','))
```

把field的值（一行多个）按照`,`（根据field值判断）分割，然后转成列

### 列转行

```sql
concat_ws(',',collect_set(column))
```

把一列值以`,`连接成一行 

把每个用户购买的品类变成一行，品类间逗号分割

```sql
SELECT user_name,
       concat_ws(',',collect_set(goods_category))
  FROM user_trade
  GROUP BY user_name;
```

### lateral view

对多字段连接

每个品类的购买用户数

```sql
SELECT b.category,
	   count(distinct a.user_name)
FROM user_goods_category a
lateral view explode(split(category_detail,',')) b as category
GROUP BY b.category;
# user_name、category_detail都在user_goods_category表里，explode相当于产生了一个新表
```

# transform

通过自定义创建新的字段

```sql
transform(旧表字段名) using "文件" as 新表字段名
```

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
# map文件，将每个词后加上1
# cat mapper.py

import sys

for line in sys.stdin:
     ss = line.strip().split(' ')
     for word in ss:
        print('%s\t1' % (word))

# reduce文件，按key计算count总数
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

 
# 建立初始表
create table docs(line string);
# 往初始表导入数据
load data local inpath '/home/badou/hive_test/The_Man_of_Property.txt' into table docs;

# 建立word count表
create table word_count(word string,count string);
# 使用初始表和map，reduce文件向word count表导入数据
insert overwrite table word_count
select transform(wc.word, wc.count) using 'python reducer.py' as word,count
from
(select transform(line) using 'python mapper.py' as word,count from docs cluster by word) wc;
```
