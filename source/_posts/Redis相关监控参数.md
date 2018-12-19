---
title: Redis相关监控参数
toc: true
date: 2018-12-19 14:30:21
tags:
 - redis
---
## 1 慢查询
默认情况下命令若是执行时间超过10ms就会被记录到日志，slowlog只会记录其命令执行的时间，不包含io往返操作，也不记录单由网络延迟引起的响应慢。如果想修改慢命令的标准可以使用下面的命令 
```
# 超过5毫秒为慢命令
config set slowlog-log-slower-than 5000
```
<!-- more -->
获取最慢的n条数据
```
slowlog get {n}
```
eg.
```
127.0.0.1:6379> slowlog get 2
1) 1) (integer) 7923
   2) (integer) 1544423728
   3) (integer) 27501
   4) 1) "KEYS"
      2) "*"

```
参数含义:
- 1=日志的唯一标识符
- 2=被记录命令的执行时间点，以 UNIX 时间戳格式表示
- 3=查询执行时间，以微秒为单位。例子中命令使用27毫秒。
- 4= 执行的命令，以数组的形式排列。完整命令是config get *。

## 2 大对象
使用scanning方式，对redis整个keyspace进行统计（数据量大时采样），寻找每种数据类型key的最大size(key)和平均size。(5种数据类型(String、hash、list、set、zset)的最大key).
 该程序使用 SCAN 命令，因此它可以在不影响客户端操作的情况下在繁忙的服务器上执行，不过也可以使用-i选项来限制所请求的每100个键的扫描过程的秒数。 例如，-i 0.1会减慢程序的执行速度，但也会大幅减轻服务器上的负载
执行命令:
```
redis-cli -h {ip} -p {port} --bigkeys
```
建议：
>把大对象拆分为多个小对象，防止一次命令操作过多数据  

eg.
```
~]# redis-cli --bigkeys

#Scanning the entire keyspace to find biggest keys as well as
#average sizes per key type.  You can use -i 0.1 to sleep 0.1 sec
#per 100 SCAN commands (not usually needed).
[00.00%] Biggest string found so far 'db:zz' with 23 bytes
[00.00%] Biggest string found so far 'prefix_1519436646796_2018_5_320401_001_9' with 73 bytes
[00.00%] Biggest string found so far 'prefix_1511407515841_2018_5_320586_101_-1' with 74 bytes
[00.00%] Biggest string found so far 'XX152844756614862018' with 1219 bytes
[00.03%] Biggest string found so far 'xxxx.m.164c2285-f8ed-4ed9-ad5e-c8633846ac95.21001004' with 24662 bytes
[00.25%] Biggest string found so far 'cpt.xml.model.key:280' with 32914 bytes
[00.71%] Biggest string found so far 'xxxx.m.164c2285-f8ed-4ed9-ad5e-c8633846ac95.48' with 94341 bytes
[01.84%] Biggest set    found so far 'spring:session:expirations:1545198360000' with 2 members
[05.58%] Biggest hash   found so far 'spring:session:sessions:53490998-30c2-4850-8f73-646fe82cd7ee' with 4 fields
[06.29%] Biggest string found so far 'xxxx.m.164c2285-f8ed-4ed9-ad5e-c8633846ac95.35001001' with 120437 bytes
[07.74%] Biggest hash   found so far 'spring:session:sessions:c28d62e7-27c5-439b-aaf7-7e3d7915db62' with 10 fields
[08.14%] Biggest string found so far 'xxxx.m.d526ca6d-f9a3-48f4-8117-3474778d1fbd.221' with 420048 bytes
[08.70%] Biggest string found so far 'xxxx.m.164c2285-f8ed-4ed9-ad5e-c8633846ac95.221' with 420086 bytes
[53.84%] Biggest set    found so far 'spring:session:expirations:1545198540000' with 4 members
[81.98%] Biggest hash   found so far 'spring:session:sessions:f3459c14-fb8d-44cc-beab-20a68a2699f0' with 11 fields
[82.04%] Biggest string found so far 'qwer' with 443204 bytes
-------- summary -------
Sampled 33464 keys in the keyspace!
Total key length in bytes is 1284216 (avg len 38.38)
Biggest string found 'qwer' has 443204 bytes
Biggest    set found 'spring:session:expirations:1545198540000' has 4 members
Biggest   hash found 'spring:session:sessions:f3459c14-fb8d-44cc-beab-20a68a2699f0' has 11 fields
33439 strings with 30379572 bytes (99.93% of keys, avg size 908.51)
0 lists with 0 items (00.00% of keys, avg size 0.00)
9 sets with 14 members (00.03% of keys, avg size 1.56)
16 hashs with 107 fields (00.05% of keys, avg size 6.69)
0 zsets with 0 members (00.00% of keys, avg size 0.00)
```

## 3 查询Redis并发量,连续统计模式
滚动显示服务器信息（keys、mem、clients、blocked、requests、connections）,默认情况下，每秒都会打印一条新的数据行，其中包含有用信息和旧数据点之间的差异 可以加-i 选项的作用就是修改输出新数据行的频率。 默认值是一秒.
```
redis-cli -h {ip} -p {port} --stat -i <interval>
```
eg.
```
~]# redis-cli --stat
------- data ------ --------------------- load -------------------- - child -
keys       mem      clients blocked requests            connections          
33463      39.28M   128     0       1597991946 (+0)     21894492    
33463      39.37M   127     0       1597991987 (+41)    21894500    
33463      39.29M   127     0       1597992021 (+34)    21894500    
33463      39.26M   127     0       1597992060 (+39)    21894500    
33463      39.37M   127     0       1597992098 (+38)    21894500
```



- 客户端相关参数  

	```
	127.0.0.1:6379> info Clients
	#Clients
	connected_clients:126
	client_longest_output_list:0
	client_biggest_input_buf:0
	blocked_clients:0
	```  

## 4 连接的客户端数量(connected_clients)
```sh
redis-cli -h {ip} -p {port} info Clients | grep connected_clients
```
这个值跟使用redis的服务的连接池配置关系比较大，Redis默认允许客户端连接的最大数量是10000。
```
127.0.0.1:6379> config get maxclients 
1) "maxclients"
2) "10000"

```
若是看到连接数超过5000以上，那可能会影响Redis的性能。倘若一些或大部分客户端发送大量的命令过来，这个数字会低的多。根据连接数负载的情况，这个数字应该设置为预期连接数峰值的110%到150之间，若是连接数超出这个数字后，Redis会拒绝并立刻关闭新来的连接。通过设置最大连接数来限制非预期数量的连接数增长，是非常重要的。
eg.
```
~]# redis-cli info Clients | grep connected_clients
connected_clients:129
```

## 5 拒绝连接数
```
redis-cli info stats | grep rejected_connections
```
eg.
```
 ~]# redis-cli info stats | grep rejected_connections
rejected_connections:0
```
## 6 阻塞客户端数量
blocked_clients，一般是执行了list数据类型的BLPOP或者BRPOP命令引起的,这个值最好应该为0
```sh
redis-cli -h {ip} -p {port} info Clients | grep blocked_clients
```
eg.
```
 ~]# redis-cli info Clients | grep blocked_clients
blocked_clients:0
```

## 7 内存碎片率
![](/image/jedis/redis-ratio.png)
```
~]# redis-cli -h {ip} -p {port} info | grep mem_fragmentation_ratio
mem_fragmentation_ratio:1.58
```
info信息中的mem_fragmentation_ratio给出了内存碎片率的数据指标，它是由操系统分配的内存除以Redis分配的内存得出：
used_memory和used_memory_rss数字都包含的内存分配有：
- 用户定义的数据：内存被用来存储key-value值。
- 内部开销： 存储内部Redis信息用来表示不同的数据类型。  

used_memory_rss的rss是Resident Set Size的缩写，表示该进程所占物理内存的大小，是操作系统分配给Redis实例的内存大小。除了用户定义的数据和内部开销以外，used_memory_rss指标还包含了内存碎片的开销，内存碎片是由操作系统低效的分配/回收物理内存导致的。 

内存碎片率稍大于1是合理的，这个值表示内存碎片率比较低，也说明redis没有发生内存交换。但如果内存碎片率超过1.5，那就说明Redis消耗了实际需要物理内存的150%，其中50%是内存碎片率。若是内存碎片率低于1的话，说明Redis内存分配超出了物理内存，操作系统正在进行内存交换。

## 8 监视在Redis中执行的命令
使用MONITOR模式后，将自动输入监控模式。它将打印Redis实例收到的所有命令
```
redis-cli -h {ip} -p {port} monitor
```
eg.
```
~]# redis-cli monitor
OK
1545199828.803965 [0 172.23.0.238:60576] "PING"
1545199828.804126 [0 172.23.0.238:60576] "SETNX" "scanKey" "xxxxxx"
1545199828.804392 [0 172.23.0.238:60576] "PING"
1545199828.804572 [0 172.23.0.238:60576] "PEXPIRE" "scanKey" "10000"
```
参考地址:
https://redis.io/topics/rediscli
https://www.cnblogs.com/mushroom/p/4738170.html