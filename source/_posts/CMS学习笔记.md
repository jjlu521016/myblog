---
title: CMS学习笔记
tags:
  - JVM
toc: true

categories:
  - java
date: 2018-12-04 23:06:59
---

# 一. 什么是cms及cms的适用场景
1. CMS ：Mostly-Concurrent收集器，也称并发标记清除收集器（Concurrent Mark-Sweep GC，CMS收集器），它管理新生代的方式与Parallel收集器和Serial收集器相同，而在老年代则是尽可能得并发执行，每个垃圾收集器周期只有2次短停顿。
2. CMS的目的： 为了消除Throught收集器和Serial收集器在Full GC周期中的`长时间停顿`。
3. CMS的使用场景：应用需要更快速的响应，不想长时间的停顿，前提条件是你的CPU资源比较丰富的条件下，适合使用CMS收集器。对于实时响应的任务，比如web server类似。

# 二. CMS的过程
- 初始标记(STW initial mark) 
- 并发标记(Concurrent marking) 
- 并发预清理(Concurrent precleaning) 
- 重新标记(STW remark) 
- 并发清理(Concurrent sweeping) 
- 并发重置(Concurrent reset)  

1. 初始标记 ：在这个阶段，需要虚拟机停顿正在执行的任务，官方的叫法STW(Stop The Word)。这个过程从垃圾回收的”根对象”开始，只扫描到能够和”根对象”直接关联的对象，并作标记。所以这个过程虽然暂停了整个JVM，但是很快就完成了。 
<img src="/image/cms/stw.png" />

2. 并发标记：由上一个阶段标记过的对象，开始tracing过程，标记所有可达的对象，这个阶段垃圾回收线程和应用线程同时运行，如上图中的黄色的点。在并发标记过程中，应用线程还在跑，因此会导致有些对象会从新生代晋升到老年代、有些老年代的对象引用会被改变、有些对象会直接分配到老年代，这些受到影响的老年代对象所在的card会被标记为dirty，用于重新标记阶段扫描。这个阶段过程中，老年代对象的card被标记为dirty的可能原因，就是下图中绿色的线：
<img src="/image/cms/Concurrent-marking.png" />