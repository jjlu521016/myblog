---
title: java中的集合学习
toc: true
date: 2017.08.31 13:30:59
tags: 
    - java
---

### java中的集合分为 value 、key-value(Collection、Map)两种
### 存储值又分为 List和Set
#### List和Set的区别
+ List是有序的,可以重复
+ Set是无序的，不可以重复的。根据equals和hashcode判断（一个对象要存储在Set中，必须重写equals和hashcode方法）。
<!-- more -->
### List常用的有ArrarList 与LinkedList(源码分析详见http://www.jianshu.com/p/0f3e65f68681)
#### ArrarList 与LinkedList的区别
+ ArrarList底层使用是数组，LinkedList使用的是链表
-  数组查询具有索引，查询特定元素比较快，而插入和删除比较慢（数组在内存中是一块连续的内存，如果插入或删除是需要移动内存）
- 链表不需要内存是连续的，在当前元素中存放下一个或上一个元素的地址，查询时需要从头部开始，一个一个的查找，所以查询效率低。插入时不需要移动内存只需要改变引用指向即可所以插入或删除的效率高。