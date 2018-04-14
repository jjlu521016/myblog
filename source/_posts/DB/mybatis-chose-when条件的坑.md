---
title: mybatis chose when条件的坑
toc: true
date: 2018-03-15 19:03:33
tags: mybatis
---

在使用mybatis 条件判断的时候，我们最常用的是: 
1. 
```xml
<if test=""></if>
```

2. 
```xml
    <choose>
        <when test="title != null">
            and title = #{title}
        </when>
        <when test="content != null">
            and content = #{content}
        </when>
        <otherwise>
            and owner = "owner1"
        </otherwise>
    </choose>
```
<!-- more -->
在编码中 我们一般习惯用
```java
if(){

} elseif(){

}else{

}
```
其中chose when otherwise等同于上面
看下面一段Mybatis代码
```xml
<choose>
    <when test="isThird == '0'">
        xxx
    </when>
    <when test="isThird == '1'">
       xxx
    </when>
    <otherwise>
    xxx
    </otherwise>
</choose>
```
不知道你有没有发现问题。对，上面代码在执行的时候死活进不去when条件，这时我们可能会说没问题啊，一定是参数传错了......
当MyBatis 判断条件为等于的时候，常量需要加 .toString() 来转换，这种方法是稳定的，推荐使用！！
