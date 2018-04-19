---
title: mysql常用SQL
toc: true
date: 2018-04-14 16:03:49
tags: 
    - mysql
---
### 1. 查询一段时间内的数据：
 查询一天：
```sql
select * from table where to_days(column_time) = to_days(now());
select * from table where date(column_time) = curdate();
```
查询一周：
<!-- more -->
```sql
select * from table   where DATE_SUB(CURDATE(), INTERVAL 7 DAY) <= date(column_time);
```
查询一个月：
```sql
select * from table where DATE_SUB(CURDATE(), INTERVAL 1 MONTH) <= date(column_time);
```
更新待续......

### 2.索引：
+ 应尽量避免在 where 子句中使用!=或<>操作符，否则将引擎放弃使用索引而进行全表扫描。

+ 对查询进行优化，应尽量避免全表扫描，首先应考虑在 where 及 order by 涉及的列上建立索引。

+ 应尽量避免在 where 子句中对字段进行 null 值判断，否则将导致引擎放弃使用索引而进行全表扫描，如：
```sql
select id from t where num is null
```
可以在num上设置默认值0，确保表中num列没有null值，然后这样查询：
```sql
select id from t where num=0
```
+ 尽量避免在 where 子句中使用 or 来连接条件，否则将导致引擎放弃使用索引而进行全表扫描，如：
```sql
select id from t where num=10 or num=20
```
可以这样查询：
```sql
select id from t where num=10
union all
select id from t where num=20
```


+ in 和 not in 也要慎用，否则会导致全表扫描。
+ 如果在 where 子句中使用参数，也会导致全表扫描。因为SQL只有在运行时才会解析局部变量，但优化程序不能将访问计划的选择推迟到运行时；它必须在编译时进行选择。然 而，如果在编译时建立访问计划，变量的值还是未知的，因而无法作为索引选择的输入项。如下面语句将进行全表扫描：
```sql
select id from t where num=@num
```
可以改为强制查询使用索引：
```sql
select id from t with(index(索引名)) where num=@num
```
+ 应尽量避免在 where 子句中对字段进行表达式操作，这将导致引擎放弃使用索引而进行全表扫描。
+ 应尽量避免在where子句中对字段进行函数操作，这将导致引擎放弃使用索引而进行全表扫描
+ 不要在 where 子句中的“=”左边进行函数、算术运算或其他表达式运算，否则系统将可能无法正确使用索引。
+ 在使用索引字段作为条件时，如果该索引是复合索引，那么必须使用到该索引中的第一个字段作为条件时才能保证系统使用该索引，否则该索引将不会被使 用，并且应尽可能的让字段顺序与索引顺序相一致。
+ 很多时候用 exists 代替 in 是一个好的选择：
+ 并不是所有索引对查询都有效，SQL是根据表中数据来进行查询优化的，当索引列有大量数据重复时，SQL查询可能不会去利用索引，如一表中有字段 sex，male、female几乎各一半，那么即使在sex上建了索引也对查询效率起不了作用。
+ 索引并不是越多越好，索引固然可以提高相应的 select 的效率，但同时也降低了 insert 及 update 的效率，因为 insert 或 update 时有可能会重建索引，所以怎样建索引需要慎重考虑，视具体情况而定。一个表的索引数最好不要超过6个，若太多则应考虑一些不常使用到的列上建的索引是否有 必要。