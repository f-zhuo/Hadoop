# wordcount

统计The_Man_of_Property.txt文件中的词频

```shell
# cat map.py 

#!/usr/local/bin/python
import sys
import time

for line in sys.stdin:
    ss = line.strip().split(' ')
    for word in ss:
        #time.sleep(100000)
        if word.strip() != "":
            print('\t'.join([word.strip(), '1']))


# cat red1.py 

#!/usr/local/bin/python
import sys
current_word = None
sum = 0

for line in sys.stdin:
    word, count = line.strip().split('\t')

    if current_word == None:
        current_word = word

    if current_word != word:
        print("%s\t%s" % (current_word, str(sum)))
        current_word = word
        sum = 0

    sum += int(count)
print("%s\t%s" % (current_word, str(sum)))


# cat red2.py 
import sys
dic={}
for line in sys.stdin:
    key=line.split('\t')[0]
    value=line.split('\t')[1]
    dic[key]=dic.get(key,0)+1

for key,value in dic.items():
    print(key+' '+str(value))


# cat run.sh 

HADOOP_CMD="/usr/local/src/hadoop-2.6.5/bin/hadoop"
STREAM_JAR_PATH="/usr/local/src/hadoop-2.6.5/share/hadoop/tools/lib/hadoop-streaming-2.6.5.jar"

INPUT_FILE_PATH_1="/The_Man_of_Property.txt"
OUTPUT_PATH="/output"

$HADOOP_CMD fs -rmr -skipTrash $OUTPUT_PATH

# \表示同一行
$HADOOP_CMD jar $STREAM_JAR_PATH \
    -input $INPUT_FILE_PATH_1 \
    -output $OUTPUT_PATH \
    -mapper "python map.py" \
    -reducer "python red1.py" \
    #-reducer "python red2.py" \
    -file ./map.py \
    -file ./red1.py
	#-file ./red2.py
```

# 分发白名单

统计白名单文件white_list在The_Man_of_Property.txt文件中的词频，white_list文件每行为一个单词

## file

```shell
# cat map.py 

#!/usr/bin/python
import sys
import time

def read_local_file_func(f):
    word_set = set()
    file_in = open(f, 'r')
    for line in file_in:
        word = line.strip()
        word_set.add(word)
    return word_set

def mapper_func(white_list_file):
    word_set = read_local_file_func(white_list_file)

    for line in sys.stdin:
        ss = line.strip().split(' ')
        for s in ss:
	    time.sleep(100)
            print "===="
            word = s.strip()
            if word != "" and (word in word_set):
				#print s + "\t" + "1"
				#print '\t'.join([s, "1"])
                print("%s\t%s" % (s, 1))


if __name__ == "__main__":
    # 模块名
    module = sys.modules[__name__]
    # 得到module的sys.argv[1]的属性，此处是获得待调用函数的函数名
    func = getattr(module, sys.argv[1]) 
    args = None
    if len(sys.argv) > 1:
        args = sys.argv[2:]
    func(*args)
    

# cat red.py 
#!/usr/bin/python
import sys

def reduer_func():
    current_word = None
    sum = 0

    for line in sys.stdin:
        word, count = line.strip().split('\t')

        if current_word == None:
            current_word = word

        if current_word != word:
            print "%s\t%s" % (current_word, sum)
            current_word = word
            sum = 0

        sum += count
        
    print("%s\t%s" % (current_word, sum))


if __name__ == "__main__":
    module = sys.modules[__name__]
    func = getattr(module, sys.argv[1])
    args = None
    if len(sys.argv) > 1:
        args = sys.argv[2:]
    func(*args)


# cat run.sh

HADOOP_CMD="/usr/local/src/hadoop-2.6.5/bin/hadoop"
STREAM_JAR_PATH="/usr/local/src/hadoop-2.6.5/contrib/streaming/hadoop-streaming-2.6.5.jar"

INPUT_FILE_PATH_1="/The_Man_of_Property.txt"
OUTPUT_PATH="/output_file_broadcast"

$HADOOP_CMD fs -rmr -skipTrash $OUTPUT_PATH

$HADOOP_CMD jar $STREAM_JAR_PATH \
    -input $INPUT_FILE_PATH_1 \
    -output $OUTPUT_PATH \
    -mapper "python map.py mapper_func white_list" \
    -reducer "python red.py reduer_func" \
    -jobconf "mapred.reduce.tasks=3" \
    -file ./map.py \
    -file ./red.py \
    -file ./white_list
```

## cachefile

```shell
# cat map.py 

#!/usr/bin/python
import sys
import time

def read_local_file_func(f):
    word_set = set()
    file_in = open(f, 'r')
    for line in file_in:
        word = line.strip()
        word_set.add(word)
    return word_set

def mapper_func(white_list_file):
    word_set = read_local_file_func(white_list_file)

    for line in sys.stdin:
        ss = line.strip().split(' ')
        for s in ss:
	    time.sleep(100)
            print "===="
            word = s.strip()
            if word != "" and (word in word_set):
				#print(s + "\t" + "1")
				#print('\t'.join([s, "1"]))
                print("%s\t%s" % (s, 1))


if __name__ == "__main__":
    module = sys.modules[__name__]
    func = getattr(module, sys.argv[1]) 
    args = None
    if len(sys.argv) > 1:
        args = sys.argv[2:]
    func(*args)
    

# cat red.py 
#!/usr/bin/python
import sys

def reduer_func():
    current_word = None
    sum = 0

    for line in sys.stdin:
        word, count = line.strip().split('\t')

        if current_word == None:
            current_word = word

        if current_word != word:
            print "%s\t%s" % (current_word, sum)
            current_word = word
            sum = 0

        sum += count
        
    print("%s\t%s" % (current_word, sum))


if __name__ == "__main__":
    module = sys.modules[__name__]
    func = getattr(module, sys.argv[1])
    args = None
    if len(sys.argv) > 1:
        args = sys.argv[2:]
    func(*args)


# cat run.sh 

HADOOP_CMD="/usr/local/src/hadoop-2.6.5/bin/hadoop"
STREAM_JAR_PATH="/usr/local/src/hadoop-2.6.5/share/hadoop/tools/lib/hadoop-streaming-2.6.5.jar"

INPUT_FILE_PATH_1="/The_Man_of_Property.txt"
OUTPUT_PATH="/output_cachefile_broadcast"

$HADOOP_CMD fs -rmr -skipTrash $OUTPUT_PATH

# Step 1.
$HADOOP_CMD jar $STREAM_JAR_PATH \
    -input $INPUT_FILE_PATH_1 \
    -output $OUTPUT_PATH \
    -mapper "python map.py mapper_func WH" \
    -reducer "python red.py reduer_func" \
    -jobconf "mapred.reduce.tasks=2" \
    -jobconf  "mapred.job.name=cachefile_demo" \
    -file "./map.py" \
    -file "./red.py" \
    -cacheFile "hdfs://master:9000/cachefile_dir/white_list.txt#WH" 
    #-cacheFile "$HDFS_FILE_PATH#WH" \
```

## cachearchive

白名单文件是压缩文件

```shell
# cat map.py 

#!/usr/bin/python
import os
import sys
import gzip

def get_file_handler(f):
    file_in = open(f, 'r')
    return file_in

def get_cachefile_handlers(f):
    f_handlers_list = []
    if os.path.isdir(f):
        for file in os.listdir(f):
            f_handlers_list.append(get_file_handler(f + '/' + file))
    return f_handlers_list

def read_local_file_func(f):
    word_set = set()
    for cachefile in get_cachefile_handlers(f):
        for line in cachefile:
            word = line.strip()
            word_set.add(word)
    return word_set

def mapper_func(white_list_fd):
    word_set = read_local_file_func(white_list_fd)

    for line in sys.stdin:
        ss = line.strip().split(' ')
        for s in ss:
            word = s.strip()
            if word != "" and (word in word_set):
                print("%s\t%s" % (s, 1))

if __name__ == "__main__":
    module = sys.modules[__name__] 
    func = getattr(module, sys.argv[1])
    args = None
    if len(sys.argv) > 1:
        args = sys.argv[2:]
    func(*args)


# cat red.py 
#!/usr/bin/python
import sys

def reduer_func():
    current_word = None
    sum = 0

    for line in sys.stdin:
        word, count = line.strip().split('\t')

        if current_word == None:
            current_word = word

        if current_word != word:
            print "%s\t%s" % (current_word, sum)
            current_word = word
            sum = 0

        sum += count
        
    print("%s\t%s" % (current_word, sum))


if __name__ == "__main__":
    module = sys.modules[__name__]
    func = getattr(module, sys.argv[1])
    args = None
    if len(sys.argv) > 1:
        args = sys.argv[2:]
    func(*args)
    
    
# cat run.sh 

HADOOP_CMD="/usr/local/src/hadoop-2.6.5/bin/hadoop"
STREAM_JAR_PATH="/usr/local/src/hadoop-2.6.5/share/hadoop/tools/lib/hadoop-streaming-2.6.5.jar"

INPUT_FILE_PATH_1="/The_Man_of_Property.txt"
OUTPUT_PATH="/output_cachearchive_broadcast"

$HADOOP_CMD fs -rmr -skipTrash $OUTPUT_PATH

$HADOOP_CMD jar $STREAM_JAR_PATH \
    -input $INPUT_FILE_PATH_1 \
    -output $OUTPUT_PATH \
    -mapper "python map.py mapper_func WH.gz" \
    -reducer "python red.py reduer_func" \
    -jobconf "mapred.reduce.tasks=2" \
    -jobconf  "mapred.job.name=cachefile_demo" \
    -cacheArchive "hdfs://master:9000/w.tar.gz#WH.gz" \
    -file "./map.py" \
    -file "./red.py"
```

# 排序

对a.txt和b.txt文件的内容排序，文件每行内容形如`1 hadoop`

MapReduce的streaming框架允许多文件输入，但对输入文件的操作都是一样的，所以本例的输入文件可以是多个待排序文件，但白名单例中的两个文件不能都做为输入文件

## 只用一个reducer

当数据源文件很大时datanode压力大

**正序排列**

逆序只需改成减法，此处对原文件中的序号做加减法是因为MR排序是按照字符逐位比较的，加/减一个较大的数保证个位数的十位，百位。。。为0

法一，不指定排序依据

```shell
# cat map_sort.py 

#!/usr/local/bin/python
import sys

base_count = 10000
#base_count = 99999 
for line in sys.stdin:
    ss = line.strip().split('\t')
    key = ss[0]
    val = ss[1]

    new_key = base_count + int(key)
     #new_key = base_count - int(key)
    print("%s\t%s" % (new_key, val))


# cat red_sort.py 
#!/usr/local/bin/python
import sys

base_count = 10000
#base_count = 99999
for line in sys.stdin:
    key, val = line.strip().split('\t')
    print(str(int(key) - base_count) + "\t" + val)
    #print(base_count -str(int(key) ) + "\t" + val)


# cat run.sh 

HADOOP_CMD="/usr/local/src/hadoop-2.6.5/bin/hadoop"
STREAM_JAR_PATH="/usr/local/src/hadoop-2.6.5/share/hadoop/tools/lib/hadoop-streaming-2.6.5.jar"

INPUT_FILE_PATH_A="/a.txt"
INPUT_FILE_PATH_B="/b.txt"

OUTPUT_SORT_PATH="/output_sort"

#$HADOOP_CMD fs -rmr -skipTrash $OUTPUT_SORT_PATH

$HADOOP_CMD jar $STREAM_JAR_PATH \
    -input $INPUT_FILE_PATH_A,$INPUT_FILE_PATH_B\
    -output $OUTPUT_SORT_PATH \
    -mapper "python map_sort.py" \
    -reducer "python red_sort.py" \
    -jobconf "mapred.reduce.tasks=1" \
    -file ./map_sort.py \
    -file ./red_sort.py \
```

法二，指定排序依据

```shell
# cat map_sort.py 
#!/usr/local/bin/python
import sys

for line in sys.stdin:
    ss = line.strip().split('\t')
    key = ss[0]
    val = ss[1]

    print "%s\t%s" % (key, val)

# cat red_sort.py 
#!/usr/local/bin/python
import sys

for line in sys.stdin:
    print line.strip()

# cat run.sh 

HADOOP_CMD="/usr/local/src/hadoop-2.6.5/bin/hadoop"
STREAM_JAR_PATH="/usr/local/src/hadoop-2.6.5/share/hadoop/tools/lib/hadoop-streaming-2.6.5.jar"

INPUT_FILE_PATH_A="/a.txt"
INPUT_FILE_PATH_B="/b.txt"

OUTPUT_SORT_PATH="/output_sort"

$HADOOP_CMD fs -rmr -skipTrash $OUTPUT_SORT_PATH

$HADOOP_CMD jar $STREAM_JAR_PATH \
    -input $INPUT_FILE_PATH_A,$INPUT_FILE_PATH_B\
    -output $OUTPUT_SORT_PATH \
    -mapper "python map_sort.py" \
    -reducer "python red_sort.py" \
    -file ./map_sort.py \
    -file ./red_sort.py \
    -partitioner org.apache.hadoop.mapred.lib.KeyFieldBasedPartitioner \
    -jobconf mapred.output.key.comparator.class=org.apache.hadoop.mapred.lib.KeyFieldBasedComparator \
    -jobconf stream.num.map.output.key.fields=1 \
    -jobconf mapred.text.key.partitioner.options="-k1,1" \
    -jobconf mapred.text.key.comparator.options="-k1,1n" \
    -jobconf mapred.reduce.tasks=1
```

## 使用两个reducer

法一，不指定排序依据

```shell
# cat map_sort.py 
#!/usr/local/bin/python
import sys

base_count = 10000

for line in sys.stdin:
    ss = line.strip().split('\t')
    key = ss[0]
    val = ss[1]

    new_key = base_count + int(key)

    red_idx = 1
    if new_key < (10100 + 10000) / 2:
        red_idx = 0

    print("%s\t%s\t%s" % (red_idx, new_key, val))


# cat red_sort.py 
#!/usr/local/bin/python
import sys

for line in sys.stdin:
    idx_id, key, val = line.strip().split('\t')
    new_key = int(key) - base_count
    print('\t'.join([new_key, val]))



# cat run.sh 

HADOOP_CMD="/usr/local/src/hadoop-2.6.5/bin/hadoop"
STREAM_JAR_PATH="/usr/local/src/hadoop-2.6.5/share/hadoop/tools/lib/hadoop-streaming-2.6.5.jar"

INPUT_FILE_PATH_A="/a.txt"
INPUT_FILE_PATH_B="/b.txt"

OUTPUT_SORT_PATH="/output_sort"

$HADOOP_CMD fs -rmr -skipTrash $OUTPUT_SORT_PATH

$HADOOP_CMD jar $STREAM_JAR_PATH \
    -input $INPUT_FILE_PATH_A,$INPUT_FILE_PATH_B\
    -output $OUTPUT_SORT_PATH \
    -mapper "python map_sort.py" \
    -reducer "python red_sort.py" \
    -file ./map_sort.py \
    -file ./red_sort.py \
    -jobconf mapred.reduce.tasks=2 \
    -jobconf stream.num.map.output.key.fields=2 \
    -jobconf num.key.fields.for.partition=1 \
    -partitioner org.apache.hadoop.mapred.lib.KeyFieldBasedPartitioner
```

法二，指定排序依据

```shell
# cat map.py 

import sys

for line in sys.stdin:
    num,course=line.strip().split()
    if int(num) < 50:
       flag=0
    else:
       flag=1
    print(str(flag)+' '+num+' '+course)


# cat red.py 
import sys

for line in sys.stdin:
     partition,num,word=line.strip().split() 
     print(num+' '+word)


# cat run.sh 

HADOOP_CMD="/usr/local/src/hadoop-2.6.5/bin/hadoop"
STREAM_JAR_PATH="/usr/local/src/hadoop-2.6.5/share/hadoop/tools/lib/hadoop-streaming-2.6.5.jar"

INPUT_FILE_PATH_1="/a.txt"
INPUT_FILE_PATH_2="/b.txt"
OUTPUT_PATH="/output_FullSort2"

$HADOOP_CMD fs -rmr -skipTrash $OUTPUT_PATH

$HADOOP_CMD jar $STREAM_JAR_PATH \
    -input $INPUT_FILE_PATH_1,$INPUT_FILE_PATH_2 \
    -output $OUTPUT_PATH \
    -mapper "python mymap2.py" \
    -reducer "python myred2.py" \
    -file ./mymap2.py \
    -file ./myred2.py \
    -jobconf stream.num.map.output.key.fields=2 \
    -jobconf num.key.fields.for.partition=1 \
    -partitioner org.apache.hadoop.mapred.lib.KeyFieldBasedPartitioner \    
    -jobconf mapred.output.key.comparator.class=org.apache.hadoop.mapred.lib.KeyFieldBasedComparator \
    -jobconf mapred.text.key.comparator.options="-k2,2n" \
    -jobconf mapred.reduce.tasks=2
```

# 压缩文件

压缩输出文件，以白名单为例

```shell
# cat map.py 

#!/usr/bin/python
import sys
import time

def read_local_file_func(f):
    word_set = set()
    file_in = open(f, 'r')
    for line in file_in:
        word = line.strip()
        word_set.add(word)
    return word_set

def mapper_func(white_list_file):
    word_set = read_local_file_func(white_list_file)

    for line in sys.stdin:
        ss = line.strip().split(' ')
        for s in ss:
	    time.sleep(100)
            print "===="
            word = s.strip()
            if word != "" and (word in word_set):
				#print(s + "\t" + "1")
				#print('\t'.join([s, "1"]))
                print("%s\t%s" % (s, 1))


if __name__ == "__main__":
    module = sys.modules[__name__]
    func = getattr(module, sys.argv[1]) 
    args = None
    if len(sys.argv) > 1:
        args = sys.argv[2:]
    func(*args)
    

# cat red.py 
#!/usr/bin/python
import sys

def reduer_func():
    current_word = None
    sum = 0

    for line in sys.stdin:
        word, count = line.strip().split('\t')

        if current_word == None:
            current_word = word

        if current_word != word:
            print "%s\t%s" % (current_word, sum)
            current_word = word
            sum = 0

        sum += count
        
    print("%s\t%s" % (current_word, sum))


if __name__ == "__main__":
    module = sys.modules[__name__]
    func = getattr(module, sys.argv[1])
    args = None
    if len(sys.argv) > 1:
        args = sys.argv[2:]
    func(*args)
    
    
# cat run.sh 

HADOOP_CMD="/usr/local/src/hadoop-1.2.1/bin/hadoop"
STREAM_JAR_PATH="/usr/local/src/hadoop-1.2.1/contrib/streaming/hadoop-streaming-1.2.1.jar"

INPUT_FILE_PATH_1="/The_Man_of_Property.txt"
OUTPUT_PATH="/output_cachearchive_broadcast"

$HADOOP_CMD fs -rmr -skipTrash $OUTPUT_PATH

$HADOOP_CMD jar $STREAM_JAR_PATH \
    -input $INPUT_FILE_PATH_1 \
    -output $OUTPUT_PATH \
    -mapper "python map.py mapper_func WH.gz" \
    -reducer "python red.py reduer_func" \
    -jobconf "mapred.reduce.tasks=10" \
    -jobconf  "mapred.job.name=cachefile_demo" \
    -jobconf  "mapred.compress.map.output=true" \
    -jobconf  "mapred.map.output.compression.codec=org.apache.hadoop.io.compress.GzipCodec" \
    -jobconf  "mapred.output.compress=true" \
    -jobconf  "mapred.output.compression.codec=org.apache.hadoop.io.compress.GzipCodec" \
    -cacheArchive "hdfs://master:9000/w.tar.gz#WH.gz" \
    -file "./map.py" \
    -file "./red.py"
```

# join表连接

对两个包含同一列内容的文件做表连接

N个map文件要分为N个步骤，前几个map过程的输出是下一个mapreduce的输入

```shell
# cat map_a.py 

#!/usr/local/bin/python
import sys

for line in sys.stdin:
    ss = line.strip().split('	')

    key = ss[0]
    val = ss[1]

    print("%s\t1\t%s" % (key, val))



# cat map_b.py 
#!/usr/local/bin/python
import sys

for line in sys.stdin:
    ss = line.strip().split('	')

    key = ss[0]
    val = ss[1]

    print("%s\t2\t%s" % (key, val))



# cat red.py 
#!/usr/local/bin/python
import sys

val_1 = ""

for line in sys.stdin:
    key, flag, val = line.strip().split('\t')

    if flag == '1':
        val_1 = val
    elif flag == '2':
        val_2 = val
        print("%s\t%s\t%s" % (key, val_1, val_2))
        val_1 = ""



# cat run.sh 

HADOOP_CMD="/usr/local/src/hadoop-2.6.5/bin/hadoop"
STREAM_JAR_PATH="/usr/local/src/hadoop-2.6.5/share/hadoop/tools/lib/hadoop-streaming-2.6.5.jar"

INPUT_FILE_PATH_A="/a.txt"
INPUT_FILE_PATH_B="/b.txt"

OUTPUT_A_PATH="/output_a"
OUTPUT_B_PATH="/output_b"

OUTPUT_JOIN_PATH="/output_join"

$HADOOP_CMD fs -rmr -skipTrash $OUTPUT_A_PATH $OUTPUT_B_PATH $OUTPUT_JOIN_PATH


# Step 1.
$HADOOP_CMD jar $STREAM_JAR_PATH \
    -input $INPUT_FILE_PATH_A \
    -output $OUTPUT_A_PATH \
    -mapper "python map_a.py" \
    -file ./map_a.py \

# Step 2.
$HADOOP_CMD jar $STREAM_JAR_PATH \
    -input $INPUT_FILE_PATH_B \
    -output $OUTPUT_B_PATH \
    -mapper "python map_b.py" \
    -file ./map_b.py \

# Step 3.
$HADOOP_CMD jar $STREAM_JAR_PATH \
    -input $OUTPUT_A_PATH,$OUTPUT_B_PATH \
    -output $OUTPUT_JOIN_PATH \
    -mapper "cat" \
    -reducer "python red.py" \
    -file ./red_join.py \
    -jobconf stream.num.map.output.key.fields=2 
    -jobconf num.key.fields.for.partition=1
```

