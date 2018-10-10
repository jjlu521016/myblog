---
title: oracle 用interval 求时间
toc: true
date: 2016-10-19 13:52:21
tags: oracle
---
```sql
select sysdate - interval '20' day as "20天前",
sysdate - interval '20' hour as "20小时前",
sysdate - interval '20' minute as "20分钟前",
sysdate - interval '20' second as "20秒钟前",
sysdate - 20 as "20天前",
sysdate - 20 / 24 as "20小时前",
sysdate - 20 / (24 * 60) as "20分钟前",
sysdate - 20 / (24 * 3600) as "20秒钟前"
from dual;
```
这里的 interval表示某段时间,格式是: interval '时间' ;

例如 interval '20' day 表示20天