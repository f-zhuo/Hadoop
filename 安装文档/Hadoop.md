# 安装java (jdk)

* 安装目录下执行

```shell
./jdk-xxx-linux-x64.bin 
```

jdk安装成功

* 查看java是否可用

```shell
./bin/java
```

* 配置环境变量

```shell
vi  ~/.bashrc
```

添加

```shell
export JAVA_HOME=/usr/local/src/jdk1.8.0_211
export CLASSPATH=.:$CLASSPATH:$JAVA_HOME/lib
export PATH=$PATH:$JAVA_HOME/bin
```

* 测试是否配置成功

```shell
java
```

* 给其他的集群节点安装java

在第一个节点中把jdk的.bash文件远程拷贝到其他节点

```shell
scp -rp jdk-xxx-linux-x64.bin ip_adress:/usr/local/src
```

然后重复前几步，如果输入java无响应，先bash一下

# Hadoop集群

* 安装目录下解压缩

```shell
tar xvzf hadoop-xxx.bin.tar.gz
```

* 进入解压的文件下并创建tmp文件

```shell
cd hadoop-xxx
mkdir tmp
```

* 进入conf下，修改masters

```shell
cd conf
vi masters
```

删除localhost改为master

修改slaves

```shell
vi slaves
```

删除localhost改为slave1，slave2

* 修改core-site.xml

   添加

```shell
<property>
        <name>hadoop.tmp.dir</name>
        <value>/usr/local/src/hadoop-2.6.5/tmp</value>
</property>
<property>
       <name>fs.default.name</name>
       <value>hdfs://192.168.107.100:9000</value>
</property>
```

* 修改mapred-site.xml文件

   添加

```shell
<property>
     <name>mapred.job.tracker</name>
     <value>http://192.168.107.100:9001</value>
</property>
```

* 修改hdfs-site.xml文件

  添加

```shell
<property>
      <name>dfs.replication</name>
      <value>3</value>
</property>
```

* 配置hadoop-env.sh

```shell
vi hadoop-env.sh
```

添加

```shell
 export JAVA_HOME=/usr/local/src/jdk1.8.0_211
```

* 配置DNS，通过主机名非IP地址访问

```shell
vi /etc/hosts
```

添加

```shell
192.168.107.100 master
192.168.107.98 slave1
192.168.107.99 slave2
```

* 修改主机名（临时生效）

```shell
hostname master
```

检验主机名

```shell
hostname
```

使主机名永久生效

```shell
vi /etc/sysconfig/network
```

添加/修改

```shell
HOSTNAME=master
```

* 远程复制Hadoop给其他节点

```shell
scp -rp hadoop-1.2.1 192.168.107.98:/usr/local/src

scp -rp hadoop-1.2.1 192.168.107.99:/usr/local/src
```

* 在其他节点上配置DNS，修改hostname为对应节点名

* 网络传输安全

```shell
setenforce 0
```

检查

```shell
getenforce
```

每个节点都要做

* 免密登陆

在master上

```shell
ssh-keygen
```

一路回车

 ```shell
cd  ~/.ssh
 ```

把公钥文件拷贝到authorized_keys

```shell
cat id_rsa.pub > authorized_keys
```

在其他两个节点上

```shell
ssh-keygen
```

一路回车

 ```shell
cd  ~/.ssh
 ```

把slave节点的公钥文件内容拷贝到master的authorized_keys里

 ```shell
scp -rp id_rsa.pub master:~/.ssh
cat id_rsa.pub >> authorized_keys
 ```

两个节点都要做

把master节点的authorized_keys文件远程拷贝到其他节点

```shell
scp -rp authorized_keys slave1:~/.ssh

scp -rp authorized_keys slave2:~/.ssh
```

测试是否可以远程连接

```shell
ssh slave1
```

**启动hadoop**

bin目录下

```shell
cd /usr/local/src/hadoop-2.6.5/bin
```

第一次启动时要先初始化

```shell
./hadoop namenode -format
```

启动

```shell
./start-all.sh
```

查看进程

```shell
jps
```

Hadoop监控页面：

http://master:8088/cluster