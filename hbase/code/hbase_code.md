# hbase的多种实践方法

## 终端

启动终端

```shell
hbase shell
```

查看数据库状态 

```shell
status
```

查看帮助 

```shell
help
```

查看有哪些表格

```shell
list
```

创建表格

```shell
create 'class', 'message', 'flag' # 表名，column family1，column family2
```

表格是否存在

```shell
exists "class"
```

表格是否激活状态

```shell
is_enabled "class"
```

把表格变成不激活状态

```shell
disable "class"
```

激活表格 

```shell
enable "class"
```

 查看表结构

```shell
describe "class"

#desc 'class'
```

查看数据（全表扫描，不建议直接用）

```shell
scan "class"
```

查看表格有多少条记录

```shell
count "class"
```

写数据

``` shell
put "class", '1001', 'message:name', 'ZhangSan' 
# 表名，row key，column family:column qualifier，数据
```

读数据

``` shell
get "class", '1001','message'
```

指定版本号读取

```shell
put "class", '1001', 'message:name', 'ZhaoSi'
put "class", '1001', 'message:name', 'WangWu'
put "class", '1001', 'message:name', 'ZhaoLiu'

get "class", '1001'

get "class", '1001', {COLUMN=>"message:name", VERSIONS => 1}
get "class", '1001', {COLUMN=>"message:name", VERSIONS => 2}
get "class", '1001', {COLUMN=>"message:name", VERSIONS => 3}

get "class", '1001', {COLUMN=>"message:name", TIMESTAMP=>1573349851782}
get "class", '1001', {COLUMN=>"message:name", TIMESTAMP=>1573349547463}
```

增加cf

```shell
alter "class", {NAME=>'contents'，VERSIONS=>3}

alter "class",'contents' # VERSIONS默认为1
```

修改cf的版本号

```shell
alter "class", {NAME=>'contents'，VERSIONS=>2}
```

删除整个cf

```shell
alter "class", {NAME=>'contents', METHOD=>'delete'}
```

删掉一条记录或数据

```shell
deleteall "class", "1001"

delete "class","1001","message:name"
```

删除表格

```shell
disable "class"

drop "class"
```

截断表/清空词表

```shell
truncate "class"
```

通过value精确匹配，反查记录

```shell
scan "class",  FILTER=>"ValueFilter(=,'binary:WangWu')"
```

通过value模糊匹配，反查记录

```shell
scan "class",  FILTER=>"ValueFilter(=,'substring:ang')"
```

多条件限制

* 列和值都有限制

```shell
scan "class", FILTER=>"ColumnPrefixFilter('na') AND ValueFilter(=, 'substring:ang')"
# column指column qualifer
```

限制rowkey的前缀

```shell
put "class", '3001', 'message:name', 'XiaoMing'

scan "class", FILTER=>"PrefixFilter('10')" 
```

指定rowkey的范围

```shell
scan "class", {STARTROW=>'1002'}

scan "class", {STARTROW=>'1002', FILTER=>"PrefixFilter('10')"}
```

正则过滤

```shell
put "class","user|4001","message:name","LiLei"


import org.apache.hadoop.hbase.filter.RegexStringComparator
import org.apache.hadoop.hbase.filter.CompareFilter
import org.apache.hadoop.hbase.filter.SubstringComparator
import org.apache.hadoop.hbase.filter.RowFilter

 
scan "class", {FILTER=>RowFilter.new(CompareFilter::CompareOp.valueOf('EQUAL'), RegexStringComparator.new('^user\|\d+$'))}
```

## 脚本

执行

```shell
hbase shell hbase_sql.sh
```

```shell
# cat hbase_sql.sh
create 'classes','user'

put 'classes','001','user:name','lilei'
put 'classes','001','user:age','20'
put 'classes','002','user:name','hanmeimei'
put 'classes','002','user:age','18'

exit # 不会自动结束
```

## python

 需要依赖模块thrift，thrift：中转，允许除了Java的其他语言开发

 ### 安装thrift

* 下载`thrift-0.8.0.tar.gz`

* 解压

```shell
tar xzf thrift-0.8.0.tar.gz
```

* 进入目录

1）configure

2）make

3）make install

* 下载`hbase-0.98.24-src.tar.gz`源码包

wget [https://mirrors.tuna.tsinghua.edu.cn/apache/hbase//0.98.24/hbase-0.98.24-src.tar.gz](https://mirrors.tuna.tsinghua.edu.cn/apache/hbase/0.98.24/hbase-0.98.24-src.tar.gz)

* 解压后，进入目录，再进入子目录` hbase-thrift/src/main/resources/org/apache/hadoop/hbase/thrift`中

* `thrift --gen py Hbase.thrift`

产生`gen-py`，目录下存在hbase的模块

 ### 启动thrift

bin目录下

```shell
hbase-daemon.sh start thrift
```

* 创建表格

`python create_table.py`

```shell
#cat create_table.py 

from thrift import Thrift
from thrift.transport import TSocket
from thrift.transport import TTransport
from thrift.protocol import TBinaryProtocol

from hbase import Hbase
from hbase.ttypes import *

transport = TSocket.TSocket('master', 9090)
transport = TTransport.TBufferedTransport(transport)

protocol = TBinaryProtocol.TBinaryProtocol(transport)
client = Hbase.Client(protocol)

transport.open()

base_info_contents = ColumnDescriptor(name='meta-data', maxVersions=1)
other_info_contents = ColumnDescriptor(name='flags', maxVersions=1)

client.createTable('new_music_table', [base_info_contents, other_info_contents])

print(client.getTableNames())
```

* 写数据

`python insert_data.py `

```shell
#cat insert_data.py 
  
from thrift import Thrift
from thrift.transport import TSocket
from thrift.transport import TTransport
from thrift.protocol import TBinaryProtocol

from hbase import Hbase
from hbase.ttypes import *

transport = TSocket.TSocket('master', 9090)
transport = TTransport.TBufferedTransport(transport)

protocol = TBinaryProtocol.TBinaryProtocol(transport)
client = Hbase.Client(protocol)

transport.open()

tableName = 'music'
rowKey = '1100'

mutations = [Mutation(column="meta-data:name", value="yueguang"), \
        Mutation(column="meta-data:tag", value="pop"), \
        Mutation(column="flags:is_valid", value="TRUE")]

client.mutateRow(tableName, rowKey, mutations, None)
```

* 读一行记录

 `python get_one_line.py `

```shell
# cat get_one_line.py
 
from thrift import Thrift
from thrift.transport import TSocket
from thrift.transport import TTransport
from thrift.protocol import TBinaryProtocol

from hbase import Hbase
from hbase.ttypes import *

transport = TSocket.TSocket('master', 9090)
transport = TTransport.TBufferedTransport(transport)

protocol = TBinaryProtocol.TBinaryProtocol(transport)
client = Hbase.Client(protocol)

transport.open()

tableName = 'music'
rowKey = '1100'

result = client.getRow(tableName, rowKey, None)

for r in result:
    print('the row is ' , r.row)
    print('the name is ' , r.columns.get('meta-data:name').value)
    print('the flag is ' , r.columns.get('flags:is_valid').value)
```

* 扫描读取多行记录

 `python scan_many_lines.py`

```shell
# cat scan_many_lines.py

from thrift import Thrift
from thrift.transport import TSocket
from thrift.transport import TTransport
from thrift.protocol import TBinaryProtocol

from hbase import Hbase
from hbase.ttypes import *

transport = TSocket.TSocket('master', 9090)
transport = TTransport.TBufferedTransport(transport)

protocol = TBinaryProtocol.TBinaryProtocol(transport)

client = Hbase.Client(protocol)

transport.open()

tableName = 'music'

scan = TScan()
id = client.scannerOpenWithScan(tableName, scan, None)
result = client.scannerGetList(id, 10) # 扫描10行

for r in result:
    print('*'*20)
    print('the row is ' , r.row)

    for k, v in r.columns.items():
        print("\t".join([k, v.value]))
```

## MR+hbase

```shell
#cat map.py

#!/usr/bin/python

import os
import sys

os.system('tar xvzf hbase.tgz > /dev/null')
os.system('tar xvzf thrift.tgz > /dev/null')

reload(sys)
sys.setdefaultencoding('utf-8')

sys.path.append("./")

from thrift import Thrift
from thrift.transport import TSocket
from thrift.transport import TTransport
from thrift.protocol import TBinaryProtocol

from hbase import Hbase
from hbase.ttypes import *

transport = TSocket.TSocket('master', 9090)
transport = TTransport.TBufferedTransport(transport)

protocol = TBinaryProtocol.TBinaryProtocol(transport)

client = Hbase.Client(protocol)

transport.open()

tableName = 'music'

def mapper_func():
    for line in sys.stdin:
        ss = line.strip().split('\t')
        if len(ss) != 2:
            continue
        key = ss[0].strip()
        val = ss[1].strip()

        rowKey = key

        mutations = [Mutation(column="meta-data:name", value=val), \
                Mutation(column="flags:is_valid", value="TRUE")]

        client.mutateRow(tableName, rowKey, mutations, None)

if __name__ == "__main__":
    module = sys.modules[__name__]
    func = getattr(module, sys.argv[1])
    args = None
    if len(sys.argv) > 1:
        args = sys.argv[2:]
    func(*args)



#cat run.sh

HADOOP_CMD="/usr/local/src/hadoop-2.6.5/bin/hadoop"
STREAM_JAR_PATH="/usr/local/src/hadoop-2.6.5/share/hadoop/tools/lib/hadoop-streaming-2.6.5.jar"

INPUT_FILE_PATH_1="/input.data_2"
OUTPUT_PATH="/output_hbase"

$HADOOP_CMD fs -rmr -skipTrash $OUTPUT_PATH

$HADOOP_CMD jar $STREAM_JAR_PATH \
    -input $INPUT_FILE_PATH_1 \
    -output $OUTPUT_PATH \
    -mapper "python map.py mapper_func" \
    -file ./map.py \
    -file "./hbase.tgz" \
    -file "./thrift.tgz"               
```

## Hive+hbase

创建hbase表

```shell
create 'classes','user'

put 'classes','001','user:name','lilei'
put 'classes','001','user:age','20'
put 'classes','002','user:name','hanmeimei'
put 'classes','002','user:age','18'
```

创建hive表

```shell
CREATE EXTERNAL Table classes(id int, name string, age int)
STORED BY 'org.apache.hadoop.hive.hbase.HBaseStorageHandler'
WITH SERDEPROPERTIES("hbase.columns.mapping"=":key,user:name,user:age")
TBLPROPERTIES("hbase.table.name"="classes")
```

返回到hbase中，插入数据测试

```shell
put 'classes','300','user:age','19'
```

