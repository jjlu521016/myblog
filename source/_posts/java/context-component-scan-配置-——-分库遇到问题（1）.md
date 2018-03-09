---
title: '<context:component-scan> 配置 —— 分库遇到问题（1）'
toc: true
date: 2017-07-26 17:37:39
tags: 
    - spring
---
项目中 springMvc的部分配置如下：
```
<dubbo:annotation />
<context:component-scan base-package="xxx">
    <context:include-filter type="annotation" expression="org.springframework.stereotype.Controller" />
    <context:include-filter type="annotation" expression="org.springframework.web.bind.annotation.RestController" />
</context:component-scan>
```
相信有些人看到我贴出来的配置就知道我要说明什么问题了，如果你还是没有头绪的话，可以看下我遇到的问题。
<!-- more -->
这个配置文件本来是想要扫描 xxx包下面的Controller和 RestControl注解，看起来并没有什么问题。我无意中一次测试发现某些service被初始化了两次！这跟spring中的单例模式是相悖的。并且一个service在spring根容器和springMvc容器分别初始化一次，导致在根容器初始化的Service里面的dubbo的 @Reference无法注入。
于是开始排查错误：除了dubbo:annotation是本人加的，其他的配置都是已经存在的。当时知道肯定是配置文件出了问题，但是不知道具体是哪里。问了公司的其他人员还是没有找到根本原因，经过反复排除并且在spring的官方文档发现了问题的根源。

### spring官方给的解释如下:
>The following example shows the configuration ignoring all @Repository annotations and using "stub" repositories instead.
```
@Configuration
@ComponentScan(basePackages = "org.example",
        includeFilters = @Filter(type = FilterType.REGEX, pattern = ".*Stub.*Repository"),
        excludeFilters = @Filter(Repository.class))
public class AppConfig {
    ...
}
```
and the equivalent using XML
```
<beans>
    <context:component-scan base-package="org.example">
        <context:include-filter type="regex"
                expression=".*Stub.*Repository"/>
        <context:exclude-filter type="annotation"
                expression="org.springframework.stereotype.Repository"/>
    </context:component-scan>
</beans>
```
[Note]
You can also disable the default filters by setting useDefaultFilters=false on the annotation or providing use-default-filters="false" as an attribute of the <component-scan/> element. This will in effect disable automatic detection of classes annotated with @Component, @Repository, @Service, @Controller, or @Configuration.

所以通过官方的描述可以看出原来是<component-scan/>使用错误导致的。也就是说如果想让项目中的<context:include-filter/>生效就必须要加use-default-filters="false" 否则 spring还是会扫描包下面的以下注解 @Component, @Repository, @Service, @Controller, or @Configuration.