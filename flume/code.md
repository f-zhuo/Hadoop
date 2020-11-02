# 本地运行

* Netcat（flume_netcat.conf）

```shell
./bin/flume-ng agent --conf conf --conf-file ./conf/flume_netcat.conf --name a1 -Dflume.root.logger=INFO,console

# 发数据
telnet master 44444
```

* exec（flume_exec.conf）

```shell
./bin/flume-ng agent --conf conf --conf-file ./conf/flume_exec.conf --name a1 -Dflume.root.logger=INFO,console

# 发数据
echo "666" >> 1.log
```

* 输出到HDFS

```shell
./bin/flume-ng agent --conf conf --conf-file ./conf/flume.conf --name a1 -Dflume.root.logger=INFO,console  

# 发数据
echo "666" >> 1.log
```

# 集群运行

* 故障转移（failover）

```shell
# master

./bin/flume-ng agent --conf conf --conf-file ./conf/agent_agent_collector_base/flume-client.properties --name agent1 -Dflume.root.logger=INFO,console

# 发送数据
for i in `seq 1 10`;do echo $i"_000" >> 1.log ;done 
 

# slave1

./bin/flume-ng agent --conf conf --conf-file ./conf/agent_agent_collector_base/flume-server.properties --name a1 -Dflume.root.logger=INFO,console


# slave2

./bin/flume-ng agent --conf conf --conf-file ./conf/agent_agent_collector_base/flume-server.properties --name a1 -Dflume.root.logger=INFO,console
```

* 负载均衡（loadbalance）

```shell
# master

./bin/flume-ng agent --conf conf --conf-file ./conf/agent_agent_collector_base/flume-client.properties_loadbalance --name a1 -Dflume.root.logger=INFO,console

# 发送数据
for i in `seq 1 10`;do echo $i"_000" >> 1.log ;done 


# slave1

./bin/flume-ng agent --conf conf --conf-file ./conf/agent_agent_collector_base/flume-server.properties --name a1 -Dflume.root.logger=INFO,console

 
# slave2

./bin/flume-ng agent --conf conf --conf-file ./conf/agent_agent_collector_base/flume-server.properties --name a1 -Dflume.root.logger=INFO,console
```

* 拦截与过滤（interceptor）

Timestamp Interceptor

```shell
./bin/flume-ng agent --conf conf --conf-file ./conf/interceptor_test/flume_ts_interceptor.conf --name a1 -Dflume.root.logger=INFO,console

# 发送数据
curl -X POST -d '[{"headers":{"hadoop1":"hadoop1 is header"}, "body":"hellobadou"}]' http://master:52020 
```

Host Interceptor

```shell
./bin/flume-ng agent --conf conf --conf-file ./conf/interceptor_test/flume_hostname_interceptor.conf --name a1 -Dflume.root.logger=INFO,console

# 发送数据
echo "abc123" | nc master 52020
```

Static Interceptor

```shell
./bin/flume-ng agent --conf conf --conf-file ./conf/interceptor_test/flume_static_interceptor.conf --name a1 -Dflume.root.logger=INFO,console

# 发送数据
curl -X POST -d '[{"headers":{"hadoop1":"hadoop1 is header"}, "body":"hellobadou"}]' http://master:52020
```

Regex Filtering Interceptor

```shell
./bin/flume-ng agent --conf conf --conf-file ./conf/interceptor_test/flume_regex_interceptor.conf --name a1 -Dflume.root.logger=INFO,console

# 发送数据
curl -X POST -d '[{"headers":{"hadoop1":"hadoop1 is header"}, "body":"hellobadou123"}]' http://master:52020
```

Regex Extractor Interceptor

```shell
./bin/flume-ng agent --conf conf --conf-file ./conf/interceptor_test/flume_regex_interceptor.conf_extractor --name a1 -Dflume.root.logger=INFO,console

# 发送数据
curl -X POST -d '[{"headers":{"hadoop1":"hadoop1 is header"}, "body":"hh1:s2:3:4:5"}]' http://master:52020
```

复制与复用（selector）

* 复制（以广播的形式发送给下游节点）

 ```shell
# master

./bin/flume-ng agent --conf conf --conf-file ./conf/selector_test/flume_client_replicating.conf --name a1 -Dflume.root.logger=INFO,console

# 发送数据
echo "good" | nc master 50000

# slave1

./bin/flume-ng agent --conf conf --conf-file ./conf/selector_test/flume_server.conf --name a1 -Dflume.root.logger=INFO,console

# slave2

./bin/flume-ng agent --conf conf --conf-file ./conf/selector_test/flume_server.conf --name a2 -Dflume.root.logger=INFO,console
 ```

* 复用

```shell
# master

./bin/flume-ng agent --conf conf --conf-file ./conf/selector_test/flume_client_multiplexing.conf --name a1 -Dflume.root.logger=INFO,console

# 发送数据
curl -X POST -d '[{"headers":{"areyouok":"OK","hadoop1":"hadoop1 is header"}, "body":"11122233"}]' http://master:50000 


# slave1

./bin/flume-ng agent --conf conf --conf-file ./conf/selector_test/flume_server.conf --name a1 -Dflume.root.logger=INFO,console


# slave2

./bin/flume-ng agent --conf conf --conf-file ./conf/selector_test/flume_server.conf --name a2 -Dflume.root.logger=INFO,console
```



