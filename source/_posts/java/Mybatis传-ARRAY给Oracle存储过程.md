---
title: Mybatis传 ARRAY给Oracle存储过程
date: 2017.03.10 23:29:38
tags: Mybatis
toc: true
---

### 1. 首先建立相关的存储过程，如图：


![11.png](http://upload-images.jianshu.io/upload_images/3353177-084719ab0ad30505.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
<!-- more -->
### 2.在service实现层填充参数 例如：
```java
@Override
	public Dto addFloorList(BannerRequest req)
	{
		Dto dto = new Dto();
		List<IndexFloor> list = new ArrayList<IndexFloor>();
		for (int i = 0; i < 3; i++)
		{
			IndexFloor flr = new IndexFloor();
			flr.setFloorId(i + 3);
			flr.setFloorTitle("测试数据 " + i + 1);
			flr.setFlrSort(i + 1);
			flr.setFlrCat(2);
			list.add(flr);
		}
		Map<String, Object> param = new HashMap<String, Object>();
		param.put("id", 2);
		param.put("floor_list", list);
//调用存储过程
		webMapper.spInsertInfo(param);
		int rtnCode = (Integer) param.get("rtnCode");
		String rtnMsg = String.valueOf(param.get("rtnMsg"));
		System.out.print(rtnCode + "------" + rtnMsg);
```

### 3.编写工具类，用来处理Array参数：

  核心代码如下（详见附件中的 handler/ArrayHandler.java）:

![22.png](http://upload-images.jianshu.io/upload_images/3353177-0c0e8e0d9bfd65bb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


（注意：上图中的IndexFloor为测试bean。具体bean对象，根据需要编写）

### 4.在MyBatis配置文件中，配置<typeHandlers>，handler为我们写好的工具类：

配置代码如下：
```xml
<?xml version="1.0" encoding="UTF-8"?>

<!DOCTYPE configuration PUBLIC "-//mybatis.org//DTD Config 3.0//EN"

 "http://mybatis.org/dtd/mybatis-3-config.dtd">

<configuration>

 <!-- 自定义传入参数类型(TypeHandler) -->

 <typeHandlers>

 <!-- 如果没有在Handler中加注解，用第一个 -->

 <!--<typeHandler javaType="list" jdbcType="ARRAY" handler="cn.hao24.api.handler.ArrayHandler"/>-->

 <typeHandler javaType="list" handler="cn.hao24.api.handler.ArrayHandler" />

 </typeHandlers>

</configuration>
```
### 5.在Mapper.xml文件中调用存储过程并传参数：
```xml
<select id="spInsertInfo" parameterType="Map" resultType="Map" statementType="CALLABLE">

 call TMP_LUJIE_TEST_ARRAY

 (

 #{rtnCode,mode=OUT,jdbcType=INTEGER,javaType=Integer},

 #{rtnMsg,mode=OUT,jdbcType=VARCHAR,javaType=String},

 #{id,mode=IN,jdbcType=INTEGER,javaType=Integer},

 **  #{floor_list,jdbcType=ARRAY,javaType=list}**

 )

 </select>
```
### 6．调试代码。

参考资料：

1.how-to-pass-java-list-of-objects-to-oracle-stored-procedure-using-mybatis：

[http://stackoverflow.com/questions/12719689/how-to-pass-java-list-of-objects-to-oracle-stored-procedure-using-mybatis](http://stackoverflow.com/questions/12719689/how-to-pass-java-list-of-objects-to-oracle-stored-procedure-using-mybatis)

2\. myBatis自定义传入参数类型(TypeHandler)

[http://jusesgod.iteye.com/blog/1740889](http://jusesgod.iteye.com/blog/1740889)
