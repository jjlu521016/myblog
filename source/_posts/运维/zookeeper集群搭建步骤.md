---
title: zookeeper集群搭建步骤
date: 2018-01-01 17:13:21
tags: 
	- zookeeper
	- 集群
---
* zookeeper服务器的数量是 `2*n+1`台
## zookeeper集群搭建步骤
本人下载目录为 /opt/microServer/ 集群ip为192.168.188.111、192.168.188.112、192.168.188.113
### 1. 下载zookeeper
执行命令下载
```sh
wget http://mirror.bit.edu.cn/apache/zookeeper/zookeeper-3.4.11/zookeeper-3.4.11.tar.gz
tar -zxvf zookeeper-3.4.11.tar.gz
cd zookeeper-3.4.11
## 创建 data和log文件夹
mkdir data log
```
### 2. 进入conf文件夹，配置zookeeper
```sh
cd conf
cp zoo_sample.cfg  zoo.cfg 
```
### 3. 编辑zoo.cf文件
```properties
## tickTime：这个时间是作为 Zookeeper 服务器之间或客户端与服务器之间维持心跳的时间间隔，也就是每个 tickTime 时间就会发送一个心跳。
tickTime=2000
## initLimit： 这个配置项是用来配置 Zookeeper 接受客户端（这里所说的客户端不是用户连接 Zookeeper 服务器的客户端，而是 Zookeeper 服务器集群中连接到 Leader 的 Follower 服务器）初始化连接时最长能忍受多少个心跳时间间隔数。当已经超过 5个心跳的时间（也就是 tickTime）长度后 Zookeeper 服务器还没有收到客户端的返回信息，那么表明这个客户端连接失败。总的时间长度就是 5*2000=10 秒
initLimit=10
## syncLimit：这个配置项标识 Leader 与Follower 之间发送消息，请求和应答时间长度，最长不能超过多少个 tickTime 的时间长度，总的时间长度就是5*2000=10秒
syncLimit=5
## dataDir：快照日志的存储路径
dataDir=/opt/microServer/zookeeper-3.4.11/data
## dataLogDir：事物日志的存储路径，如果不配置这个那么事物日志会默认存储到dataDir制定的目录，这样会严重影响zk的性能，当zk吞吐量较大的时候，产生的事物日志、快照日志太多
dataLogDir=/opt/microServer/zookeeper-3.4.11/log
clientPort=2181
server.1=192.168.188.111:12888:13888
server.2=192.168.188.112:12888:13888
server.3=192.168.188.113:12888:13888
#server.1 这个1是服务器的标识也可以是其他的数字， 表示这个是第几号服务器，用来标识服务器，这个标识要写到快照目录下面myid文件里
#192.168.188.111为集群里的IP地址，第一个端口是master和slave之间的通信端口，默认是2888，第二个端口是leader选举的端口，集群刚启动的时候选举或者leader挂掉之后进行新的选举的端口默认是3888
```

### 4.复制zookeeper到其他机器
```sh
scp -r zookeeper-3.4.11/ root@192.168.188.112:/opt/microServer/
scp -r zookeeper-3.4.11/ root@192.168.188.113:/opt/microServer/
```
### 4. 创建myid文件
``` sh
#server1
echo "1" > /opt/microServer/zookeeper-3.4.11/data/myid
#server2
echo "2" > /opt/microServer/zookeeper-3.4.11/data/myid
#server3
echo "3" > /opt/microServer/zookeeper-3.4.11/data/myid
```
### 启动服务并查看(可以在/etc/profile配置zk)
```sh
#进入到Zookeeper的bin目录下
cd /opt/zookeeper/zookeeper-3.4.6/bin
#启动服务（3台都需要操作）
./zkServer.sh start
#检查服务器状态
./zkServer.sh status
```
![1.png](http://upload-images.jianshu.io/upload_images/3353177-5f3aa2c07079af7e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![2.png](http://upload-images.jianshu.io/upload_images/3353177-79931c930bf42288.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![3.png](http://upload-images.jianshu.io/upload_images/3353177-ab82da0571ad20f4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
### *注意事项：
如果zkServer.sh status出现以下错误：
```log
]# zkServer.sh status
ZooKeeper JMX enabled by default
Using config: /opt/microserver/zookeeper/bin/../conf/zoo.cfg
Error contacting service. It is probably not running.
```
到 /root/zookeeper.out看报错信息。如下：
 ```log
 [myid:1] - WARN  [QuorumPeer[myid=1]/0:0:0:0:0:0:0:0:2181:Learner@237] - Unexpected exception, tries=0, connecting to /192.168.188.113:12888
java.net.ConnectException: 拒绝连接 (Connection refused)
	at java.net.PlainSocketImpl.socketConnect(Native Method)
	at java.net.AbstractPlainSocketImpl.doConnect(AbstractPlainSocketImpl.java:350)
	at java.net.AbstractPlainSocketImpl.connectToAddress(AbstractPlainSocketImpl.java:206)
	at java.net.AbstractPlainSocketImpl.connect(AbstractPlainSocketImpl.java:188)
	at java.net.SocksSocketImpl.connect(SocksSocketImpl.java:392)
	at java.net.Socket.connect(Socket.java:589)
	at org.apache.zookeeper.server.quorum.Learner.connectToLeader(Learner.java:229)
	at org.apache.zookeeper.server.quorum.Follower.followLeader(Follower.java:72)
	at org.apache.zookeeper.server.quorum.QuorumPeer.run(QuorumPeer.java:981)
2018-01-14 20:35:45,726 [myid:1] - INFO  [QuorumPeer[myid=1]/0:0:0:0:0:0:0:0:2181:Learner@332] - Getting a diff from the leader 0x0
```
这种情况需要关闭防火墙,执行
```sh
systemctl stop firewalld.service #停止firewall
systemctl disable firewalld.service #禁止firewall开机启动
firewall-cmd --state #查看默认防火墙状态（关闭后显示notrunning，开启后显示running）
```

