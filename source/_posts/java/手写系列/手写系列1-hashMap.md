---
title: 手写系列1-hashMap
toc: true
date: 2018-12-13 18:59:49
tags: 
    - hashMap
---

# 1.什么是HashMap
HashMap 主要用来存放键值对，它基于哈希表的Map接口实现。
在JDK1.8之前，HashMap是由数组+链表(当hash冲突时转换成链表)，其中HashMap的主体是数组。
在JDK1.8之后，当发生hash冲突的时候有了新的变化：当链表的长度大于8（默认的阈值），会将链表转换成红黑树，用来减少搜索的时间。