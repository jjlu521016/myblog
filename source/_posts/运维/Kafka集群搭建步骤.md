---
title: Kafka集群搭建步骤
date: 2018-03-07 17:05:51
tags: Kafka
---

### 1. 下载kafka(scale编译版本不同，有不同的选择)
```sh
wget https://www.apache.org/dyn/closer.cgi?path=/kafka/1.0.0/kafka_2.11-1.0.0.tgz
### 解压
tar zxvf kafka_2.11-1.0.0.tgz
### 切换到kafka目录创建日志目录
cd kafka_2.11-1.0.0
mkdir logs
```

### 2. 配置`server.properties`
```sh
cd config
vi server.properties
```
修改配置文件(按需修改)
```properties
# 当前机器在集群中的唯一标识，和zookeeper的myid性质一样
broker.id=0  
# 当前kafka对外提供服务的端口默认是9092
# port=9092 
# 参数默认是关闭的，在0.8.1有个bug，DNS解析问题，失败率的问题。
# host.name=192.168.188.111 
listeners=PLAINTEXT://192.168.188.111:9092
# borker进行网络处理的线程数
num.network.threads=3 
# borker进行I/O处理的线程数
num.io.threads=8 
# 消息存放的目录，这个目录可以配置为“，”逗号分割的表达式，上面的num.io.threads要大于这个目录的个数这个目录，如果配置多个目录，新创建的topic他把消息持久化的地方是，当前以逗号分割的目录中，那个分区数最少就放那一个
log.dirs=/opt/microServer/kafka_2.11-1.0.0/logs
# 发送缓冲区buffer大小，数据不是一下子就发送的，先回存储到缓冲区了到达一定的大小后在发送，能提高性能
socket.send.buffer.bytes=102400 
# kafka接收缓冲区大小，当数据到达一定大小后在序列化到磁盘
socket.receive.buffer.bytes=102400 
 # 向kafka请求消息或者向kafka发送消息的请请求的最大数，这个值不能超过java的堆栈大小
socket.request.max.bytes=104857600
# 默认的分区数，一个topic默认1个分区数
num.partitions=1
# 默认消息的最大持久化时间，168小时，7天
log.retention.hours=168
# 消息保存的最大值5M
message.max.byte=5242880  
# kafka保存消息的副本数，如果一个副本失效了，另一个还可以继续提供服务
default.replication.factor=2 
# 取消息的最大直接数 
replica.fetch.max.bytes=5242880  
# kafka的消息是以追加的形式落地到文件，当超过这个值的时候，kafka会新起一个文件
log.segment.bytes=1073741824 
# 每隔300000毫秒去检查上面配置的log失效时间（log.retention.hours=168 ），到目录查看是否有过期的消息如果有，删除
log.retention.check.interval.ms=300000 
# 是否启用log压缩，一般不用启用，启用的话可以提高性能
log.cleaner.enable=false
# 设置zookeeper的连接端口
zookeeper.connect=192.168.188.111:2181,192.168.188.112:2181,192.168.188.113:2181
```

### 3. 启动Kafka集群并测试（zk开启）
* 启动服务
```sh
进入kafka bin目录
##后台守护启动
bin]# ./kafka-server-start.sh -daemon ../config/server.properties
```
* 检查服务是否启动
```sh
# jps
2241 QuorumPeerMain
3236 Jps
3100 Kafka
```
* 创建Topic来验证是否创建成功
```sh
bin]# ./kafka-topics.sh --create --zookeeper 192.168.188.112:2181 --replication-factor 2 --partitions 1 --topic test
#创建一个broker，发布者
bin]# ./kafka-console-producer.sh --broker-list 192.168.188.112:9092 --topic test2
'''在一台服务器上创建一个订阅者'''
bin]# ./kafka-console-consumer.sh --zookeeper 192.168.188.112:2181 --topic test2 --from-beginning
```
测试结果
```
## server1
bin]# ./kafka-console-producer.sh --broker-list 192.168.188.112:9092 --topic test2
>werwe
>hahaha
>

##server2 
bin]# ./kafka-console-consumer.sh --zookeeper 192.168.188.112:2181 --topic test2 --from-beginning
Using the ConsoleConsumer with old consumer is deprecated and will be removed in a future major release. Consider using the new consumer by passing [bootstrap-server] instead of [zookeeper].
werwe
hahaha
```
