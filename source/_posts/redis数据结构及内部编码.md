---
title: redis数据结构及内部编码
tags: []
toc: true
originContent: ''
categories: []
date: 2019-09-16 19:49:43
---

在redis中，当我们想要知道一个key的类型的时候，我们可以使用type命令
eg
```sh
127.0.0.1:6379> set a "123"
OK
127.0.0.1:6379> type a
string
```
如果这个key不存在的话，会返回none
eg:
```sh
127.0.0.1:6379> type abcd
none
```

type命令实际返回的就是当前键的数据结构类型，它们分别是：
- string（字符串）
- hash（哈希）
- list（列表）
- set（集合）
- zset（有序集合）
但这些只是Redis对外的数据结构。每种数据结构都有自己底层的内部实现，并且每个都有多种实现，这样方便redis在合适的场景选择适合当前的编码方式。
下图是redis每种数据结构对应的内部编码
