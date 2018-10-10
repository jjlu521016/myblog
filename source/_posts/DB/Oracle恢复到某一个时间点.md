---
title: Oracle恢复到某一个时间点
toc: true
date: 2017-03-24 13:55:44
tags:
    - oracle
---
```sql
flashback table tb_goods_sku to timestamp to_timestamp('2016-04-29 12:12:12','yyyy-mm-dd hh24:mi:ss');
```
```sql
alter table tb_goods_sku enable row movement;
```
操作数据库一不小心将很重要的数据删除了，找备份也没有，幸好Oracle有闪回的功能。
<!-- more -->
```sql
Flashback table pb_acc_user  to timestamp to_timestamp
('2014-0315 09:30:00','yyyy-mm-dd hh24:mi:ss');
```
提示ORA-08189: 因为未启用行移动功能, 不能闪回表 。一般来说出现这种错误，就是数据库表不支持闪回功能，修复很简单，开启即可。
所以执行以下语句 再执行闪回.
```sql
alter table pb_acc_user enable row movement;
```
成功闪回修改.