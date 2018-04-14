---
title: Redis单个key存储的value很大
toc: true
date: 2018-03-09 14:53:18
tags:
    - redis
---

Redis使用过程中经常会有各种大key的情况， 比如：

+ 单个简单的key存储的value很大
+ hash， set，zset，list 中存储过多的元素（以万为单位）
由于redis是单线程运行的，如果一次操作的value很大会对整个redis的响应时间造成负面影响，所以，业务上能拆则拆，下面举几个典型的分拆方案。
<!-- more -->
### 1. 单个简单的key存储的value很大
#### 1.1 改对象需要每次都整存整取
可以尝试将对象分拆成几个key-value， 使用multiGet获取值，这样分拆的意义在于分拆单次操作的压力，将操作压力平摊到多个redis实例中，降低对单个redis的IO影响；    

#### 1.2 该对象每次只需要存取部分数据
可以像第一种做法一样，分拆成几个key-value，  也可以将这个存储在一个hash中，每个field代表一个具体的属性，使用hget,hmget来获取部分的value，使用hset，hmset来更新部分属性    

### 2. hash、set、zset、list 中存储过多的元素
类似于场景一种的第一个做法，可以将这些元素分拆。

以hash为例，原先的正常存取流程是  hget(hashKey, field) ; hset(hashKey, field, value) 
现在，固定一个桶的数量，比如 10000， 每次存取的时候，先在本地计算field的hash值，模除 10000， 确定了该field落在哪个key上。
```java
newHashKey  =  hashKey + (*hash*(field) % 10000）;   
hset (newHashKey, field, value) ;  
hget(newHashKey, field)
```
set, zset, list 也可以类似上述做法.

但有些不适合的场景，比如，要保证 lpop 的数据的确是最早push到list中去的，这个就需要一些附加的属性，或者是在 key的拼接上做一些工作（比如list按照时间来分拆）。
