---
title: jvm参数配置笔记
date: 2018.01.15 18:25:49
tags: JVM
---
### 1、堆设置（这里是重点）
-Xms:初始堆大小
-Xmx:最大堆大小
-XX:NewSize=n:设置年轻代初始大小
-XX:MaxNewSize=n设置年轻代最大大小
-Xmn:相当于-XX:NewSize和-XX:MaxNewSize设置为同一个值了，表示永久设置年轻代的大小。当然剩下的就是老年代咯。
<!-- more -->
-XX:NewRatio=n:设置年轻代和年老代的比值。如:为3，表示年轻代与年老代比值为1：3，年轻代占整个年轻代年老代和的1/4
-XX:SurvivorRatio=n:年轻代中Eden区与两个Survivor区的比值。注意Survivor区有两个。如：3，表示Eden：Survivor=3：2，一个Survivor区占整个年轻代的1/5

### 2、方法区设置(重点)
-XX:PermSize=n：设置永久代初始大小
-XX:MaxPermSize=n:设置永久代最大大小

### 3、收集器设置（java支持多种垃圾收集器，当然你或许连收集器的概念也不知道，赶紧看书去吧）
-XX:+UseSerialGC:设置串行收集器
-XX:+UseParallelGC:设置并行收集器
-XX:+UseParalledlOldGC:设置并行年老代收集器
-XX:+UseConcMarkSweepGC:设置并发收集器

### 4、垃圾回收统计信息（这个可以用于调试时候查看GC的情况）
-XX:+PrintGC
-XX:+PrintGCDetails
-XX:+PrintGCTimeStamps
-Xloggc:filename

### 5、并行收集器设置
-XX:ParallelGCThreads=n:设置并行收集器收集时使用的CPU数。并行收集线程数。
-XX:MaxGCPauseMillis=n:设置并行收集最大暂停时间
-XX:GCTimeRatio=n:设置垃圾回收时间占程序运行时间的百分比。公式为1/(1+n)

### 6、并发收集器设置
-XX:+CMSIncrementalMode:设置为增量模式。适用于单CPU情况。

### 7、栈区设置（一般甚少用到，放最后）
-Xss：每个线程的Stack大小，“-Xss 15120” 这使每增加一个线程（thread)就会立即消耗15M内存，而最佳值应该是128K,默认值好像是512k. 