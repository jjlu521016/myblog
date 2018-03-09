---
title: Redis常见问题
toc: true
date: 2018-03-09 12:02:21
tags: redis
---
### 1. 分布式锁
说道Redis分布式锁，我们的第一印象就是setnx操作。
先拿setnx来争抢锁，抢到之后，再用expire给锁加一个过期时间防止锁忘记了释放。但是在setnx之后执行expire之前进程意外crash或者要重启维护了，那会怎么样？
<!-- more -->
从 Redis 2.6.12 版本开始， SET 命令的行为可以通过一系列参数来修改：
+ EX second ：设置键的过期时间为 second 秒。 SET key value EX second 效果等同于 SETEX key second value 。
+ PX millisecond ：设置键的过期时间为 millisecond 毫秒。 SET key value PX millisecond 效果等同于 PSETEX key millisecond value 。
+ NX ：只在键不存在时，才对键进行设置操作。 SET key value NX 效果等同于 SETNX key value 。
+ XX ：只在键已经存在时，才对键进行设置操作。
所以set指令有非常复杂的参数，可以同时把setnx和expire合成一条指令来用！
参考：http://redisdoc.com/string/set.html

### 2. 无阻塞取数据
+ 假如Redis里面有1亿个key，其中有10w个key是以某个固定的已知的前缀开头的，如果将它们全部找出来？

一般人会说：使用keys指令可以扫出指定模式的key列表。
+ 如果这个redis正在给线上的业务提供服务，那使用keys指令会有什么问题？

因为redis的单线程的,keys指令会导致线程阻塞一段时间，线上服务会停顿，直到指令执行完毕，服务才能恢复。这个时候可以使用scan指令，scan指令可以无阻塞的提取出指定模式的key列表，但是会有一定的重复概率，在客户端做一次去重就可以了，但是整体所花费的时间会比直接用keys指令长。

### 3. 大量的key同一时间过期
如果大量的key过期时间设置的过于集中，到过期的那个时间点，redis可能会出现短暂的卡顿现象。一般需要在时间上加一个随机值，使得过期时间分散一些。

### 4. Redis做持久化
+ Redis如何做持久化的？

bgsave做镜像全量持久化，aof做增量持久化。因为bgsave会耗费较长时间，不够实时，在停机的时候会导致大量丢失数据，所以需要aof来配合使用。在redis实例重启时，会使用bgsave持久化文件重新构建内存，再使用aof重放近期的操作指令来实现完整恢复重启之前的状态。

+ 如果突然机器掉电会怎样？

取决于aof日志sync属性的配置，如果不要求性能，在每条写指令时都sync一下磁盘，就不会丢失数据。但是在高性能的要求下每次都sync是不现实的，一般都使用定时sync，比如1s1次，这个时候最多就会丢失1s的数据。

+ bgsave的原理是什么？

fork和cow。fork是指redis通过创建子进程来进行bgsave操作，cow指的是copy on write，子进程创建后，父子进程共享数据段，父进程继续提供读写服务，写脏的页面数据会逐渐和子进程分离开来。
