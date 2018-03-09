---
title: Spring Boot配置文件放在jar外部
toc: true
date: 2017-06-28 09:46:32
tags:
    - springboot
---
Spring Boot程序默认从application.properties或者application.yaml读取配置，如何将配置信息外置，方便配置呢？
查询官网，可以得到下面的几种方案:
### 1. 通过命令行指定
SpringApplication会默认将命令行选项参数转换为配置信息
例如，启动时命令参数指定：
```
java -jar demo.jar --server.port = 9011
```
<!-- more -->
### 2.外置配置文件

Spring程序会按优先级从下面这些路径来加载application.properties配置文件
当前目录下的/config目录
当前目录
classpath里的/config目录
classpath 跟目录
因此，要外置配置文件就很简单了，在jar所在目录新建config文件夹，然后放入配置文件，或者直接放在配置文件在jar目录.

### 3.自定义配置文件

如果你不想使用application.properties作为配置文件，怎么办？完全没问题
```
java -jar demo.jar --spring.config.location=classpath:/default.properties,classpath:/override.properties
```
或者
```
java -jar -Dspring.config.location=D:\config\config.properties demo.jar 
```
当然，还能在代码里指定
```
@SpringBootApplication
@PropertySource(value={"file:config.properties"})
public class SpringbootrestdemoApplication {
    public static void main(String[] args) {
        SpringApplication.run(SpringbootrestdemoApplication.class, args);
    }
}
```