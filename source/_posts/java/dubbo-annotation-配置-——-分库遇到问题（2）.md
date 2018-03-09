---
title: '<dubbo:annotation >配置 —— 分库遇到问题（2）'
toc: true
date: 2017-07-26 17:37:36
tags:
    - dubbo
---
在上篇笔记《<context:component-scan> 配置 —— 分库遇到问题（1）》中解决了 spring中某些实例被初始化了两次的问题，
但是紧接着又来了另一个头疼的问题，dubbo的@Reference为null无法注入 ！
Controller层的注解正常！！
```
<dubbo:annotation />
	<context:component-scan base-package="xxx" use-default-filters="false">
		<context:include-filter type="annotation" expression="org.springframework.stereotype.Controller" />
		<context:include-filter type="annotation" expression="org.springframework.web.bind.annotation.RestController" />
	</context:component-scan>

```
<!-- more -->
我把这个问题提出来之后，大家都提出来，在dubbo的service或者spring还没有初始化完成之前就开始扫描 @Reference导致取到null值。但是怎么去找到问题的根源呢？
于是几个人在一块排查，刚开始是修改spring的配置文件。把有关扫描的配置重新检查了一下，并没有发现问题。网上查关于dubbo初始化的资料，依然没有发现解决问题的方法！
查询无果后，开始往源码上面去研究。
我始终在想，之前dubbo使用没有问题的，就在我昨天加了use-default-filters="false"才出现的这个问题，所以我围绕着 context:component-scan + dubbo:annotation寻找答案，其中一条结果是指向 Dubbo的官方文档。如下：
> 服务提供方注解：
```
import com.alibaba.dubbo.config.annotation.Service;
@Service(version="1.0.0")
public class FooServiceImpl implements FooService {
}
```
服务提供方配置：
```
<!-- 公共信息，也可以用dubbo.properties配置 -->
<dubbo:application name="annotation-provider" />
<dubbo:registry address="127.0.0.1:4548" />
<!-- 扫描注解包路径，多个包用逗号分隔，不填pacakge表示扫描当前ApplicationContext中所有的类 -->
<dubbo:annotation package="com.foo.bar.service" />
```
服务消费方注解：
```
import com.alibaba.dubbo.config.annotation.Reference;
import org.springframework.stereotype.Component;
@Component
public class BarAction {
    @Reference(version="1.0.0")
    private FooService fooService;
}
```
服务消费方配置：
```
<!-- 公共信息，也可以用dubbo.properties配置 -->
<dubbo:application name="annotation-consumer" />
<dubbo:registry address="127.0.0.1:4548" />
<!-- 扫描注解包路径，多个包用逗号分隔，不填pacakge表示扫描当前ApplicationContext中所有的类 -->
<dubbo:annotation package="com.foo.bar.action" />
```
### 也可以使用：(等价于前面的：<dubbo:annotation package="com.foo.bar.service" />)
```
<dubbo:annotation />
<context:component-scan base-package="com.foo.bar.service">
    <context:include-filter type="annotation" expression="com.alibaba.dubbo.config.annotation.Service" />
</context:component-scan>
```


从官方给的样例找到了问题产生的原因。dubbo:annotation不指定包名的话会在spring bean中查找对应实例的类配置了dubbo注解的。