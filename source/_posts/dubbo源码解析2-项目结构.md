---
title: dubbo源码解析2-项目结构
tags:
  - dubbo
toc: true
originContent: >
  > 特别说明，基于2.7.0  


  下面的内容主要来自官网，可以移步[官网](http://dubbo.apache.org/zh-cn/docs/dev/design.html)

  # 1. 框架设计

  <!-- more -->

  ![image.png](/images/2019/02/15/d4cbdfa0-30dc-11e9-ade1-a511bd99bf1c.png)

  图例说明：


  - 图中左边淡蓝背景的为服务消费方使用的接口，右边淡绿色背景的为服务提供方使用的接口，位于中轴线上的为双方都用到的接口。

  - 图中从下至上分为十层，各层均为单向依赖，右边的黑色箭头代表层之间的依赖关系，每一层都可以剥离上层被复用，其中，Service 和 Config 层为
  API，其它各层均为 SPI。

  - 图中绿色小块的为扩展接口，蓝色小块为实现类，图中只显示用于关联各层的实现类。

  -
  图中蓝色虚线为初始化过程，即启动时组装链，红色实线为方法调用过程，即运行时调时链，紫色三角箭头为继承，可以把子类看作父类的同一个节点，线上的文字为调用的方法。


  # 2.模块分包

  ![image.png](/images/2019/02/15/92a6e3e0-30dc-11e9-ade1-a511bd99bf1c.png)

  ![image.png](/images/2019/02/15/21878a20-30dc-11e9-ade1-a511bd99bf1c.png)

  模块说明：


  ## 2.1 dubbo-common 

  公共逻辑模块：包括 Util 类和通用模型。

  项目代码结构如下

  ![image.png](/images/2019/02/15/365c11e0-30dd-11e9-ade1-a511bd99bf1c.png)  


  ## 2.2 dubbo-remoting 

  远程通讯模块：相当于 Dubbo 协议的实现，如果 RPC 用 RMI协议则不需要使用此包。

  ![image.png](/images/2019/02/15/c02f9630-30ec-11e9-849a-8da18d248662.png)

  - dubbo-remoting-zookeeper 
   Zookeeper 客户端，与ZK 服务器通信
  - dubbo-remoting-api

  定义Dubbo Client和 Dubbo Server的接口规则

  - dubbo-remoting-grizzly

  基于 [Grizzly](https://javaee.github.io/grizzly/) 实现。

  - dubbo-remoting-http 

  基于 [Jetty](https://www.eclipse.org/jetty/) 或
  [Tomcat](http://tomcat.apache.org/) 实现。

  - dubbo-remoting-mina 

  基于 [Mina](https://mina.apache.org/) 实现。

  - dubbo-remoting-netty 

  基于 [Netty 3](https://netty.io/) 实现。

  - dubbo-remoting-netty4 

  基于 [Netty 4](https://netty.io/) 实现。

  - dubbo-remoting-p2p 

  P2P 服务器。注册中心 `dubbo-registry-multicast` 项目的使用该项目。


  ## 2.3 dubbo-rpc 

  远程调用模块：抽象各种协议，以及动态代理，只包含一对一的调用，不关心集群的管理。

  官方性能报告地址（基于2.0版本，不是最新）：[dubbo性能测试报告](http://dubbo.apache.org/zh-cn/docs/user/perf-test.html)

  ![image.png](/images/2019/02/15/8d78d700-30ed-11e9-849a-8da18d248662.png)

  - dubbo-rpc-api

  抽象各种协议以及动态代理，实现了一对一的调用。

  - 其他模块，实现 dubbo-rpc-api ，提供对应的协议实现

  协议参考手册


  ## 2.4 dubbo-cluster 

  集群模块：将多个服务提供方伪装为一个提供方，包括：负载均衡, 容错，路由等，集群的地址列表可以是静态配置的，也可以是由注册中心下发。

  ## 2.5 dubbo-registry 

  注册中心模块：基于注册中心下发地址的集群方式，以及对各种注册中心的抽象。

  ## 2.6 dubbo-monitor 

  监控模块：统计服务调用次数，调用时间的，调用链跟踪的服务。

  ## 2.7 dubbo-config 

  配置模块：是 Dubbo 对外的 API，用户通过 Config 使用Dubbo，隐藏 Dubbo 所有细节。

  dubbo也提供了四种配置方式，

  - XML配置

  - 属性配置

  - API配置

  - 注解配置

  项目代码结构如下

  ![image.png](/images/2019/02/15/aa17a810-30e2-11e9-ade1-a511bd99bf1c.png)


  dubbo-config-api：实现了API配置和属性配置的功能。

  dubbo-config-spring：实现了XML配置和注解配置的功能


  ## 2.8 dubbo-container 

  容器模块：是一个 Standlone 的容器，以简单的 Main 加载 Spring 启动，因为服务通常不需要 Tomcat/JBoss 等 Web
  容器的特性，没必要用 Web 容器去加载服务。
categories:
  - dubbo
date: 2019-02-15 12:37:38
---

> 特别说明，基于2.7.0  

下面的内容主要来自官网，可以移步[官网](http://dubbo.apache.org/zh-cn/docs/dev/design.html)
# 1. 框架设计
<!-- more -->
![image.png](/images/2019/02/15/d4cbdfa0-30dc-11e9-ade1-a511bd99bf1c.png)
图例说明：

- 图中左边淡蓝背景的为服务消费方使用的接口，右边淡绿色背景的为服务提供方使用的接口，位于中轴线上的为双方都用到的接口。
- 图中从下至上分为十层，各层均为单向依赖，右边的黑色箭头代表层之间的依赖关系，每一层都可以剥离上层被复用，其中，Service 和 Config 层为 API，其它各层均为 SPI。
- 图中绿色小块的为扩展接口，蓝色小块为实现类，图中只显示用于关联各层的实现类。
- 图中蓝色虚线为初始化过程，即启动时组装链，红色实线为方法调用过程，即运行时调时链，紫色三角箭头为继承，可以把子类看作父类的同一个节点，线上的文字为调用的方法。

# 2.模块分包
![image.png](/images/2019/02/15/92a6e3e0-30dc-11e9-ade1-a511bd99bf1c.png)
![image.png](/images/2019/02/15/21878a20-30dc-11e9-ade1-a511bd99bf1c.png)
模块说明：

## 2.1 dubbo-common 
公共逻辑模块：包括 Util 类和通用模型。
项目代码结构如下
![image.png](/images/2019/02/15/365c11e0-30dd-11e9-ade1-a511bd99bf1c.png)  

## 2.2 dubbo-remoting 
远程通讯模块：相当于 Dubbo 协议的实现，如果 RPC 用 RMI协议则不需要使用此包。
![image.png](/images/2019/02/15/c02f9630-30ec-11e9-849a-8da18d248662.png)
- dubbo-remoting-zookeeper 
 封装了Zookeeper Client ，和 Zookeeper Server 通信
- dubbo-remoting-api
定义Dubbo Client和 Dubbo Server的接口规则
- dubbo-remoting-grizzly
基于 [Grizzly](https://javaee.github.io/grizzly/) 实现。
- dubbo-remoting-http 
基于 [Jetty](https://www.eclipse.org/jetty/) 或 [Tomcat](http://tomcat.apache.org/) 实现。
- dubbo-remoting-mina 
基于 [Mina](https://mina.apache.org/) 实现。
- dubbo-remoting-netty 
基于 [Netty 3](https://netty.io/) 实现。
- dubbo-remoting-netty4 
基于 [Netty 4](https://netty.io/) 实现。
- dubbo-remoting-p2p 
P2P 服务器。注册中心 `dubbo-registry-multicast` 项目的使用该项目。

## 2.3 dubbo-rpc 
远程调用模块：抽象各种协议，以及动态代理，只包含一对一的调用，不关心集群的管理。
官方性能报告地址（基于2.0版本，不是最新）：[dubbo性能测试报告](http://dubbo.apache.org/zh-cn/docs/user/perf-test.html)
![image.png](/images/2019/02/15/8d78d700-30ed-11e9-849a-8da18d248662.png)
- dubbo-rpc-api
抽象各种协议以及动态代理，实现了一对一的调用。
- 其他模块，实现 dubbo-rpc-api ，提供对应的协议实现
协议参考手册

## 2.4 dubbo-cluster 
集群模块：将多个服务提供方伪装为一个提供方，包括：负载均衡, 容错，路由等，集群的地址列表可以是静态配置的，也可以是由注册中心下发。
## 2.5 dubbo-registry 
注册中心模块：基于注册中心下发地址的集群方式，以及对各种注册中心的抽象。
## 2.6 dubbo-monitor 
监控模块：统计服务调用次数，调用时间的，调用链跟踪的服务。
## 2.7 dubbo-config 
配置模块：是 Dubbo 对外的 API，用户通过 Config 使用Dubbo，隐藏 Dubbo 所有细节。
dubbo也提供了四种配置方式，
- XML配置
- 属性配置
- API配置
- 注解配置
项目代码结构如下
![image.png](/images/2019/02/15/aa17a810-30e2-11e9-ade1-a511bd99bf1c.png)

dubbo-config-api：实现了API配置和属性配置的功能。
dubbo-config-spring：实现了XML配置和注解配置的功能

## 2.8 dubbo-container 
容器模块：是一个 Standlone 的容器，以简单的 Main 加载 Spring 启动，因为服务通常不需要 Tomcat/JBoss 等 Web 容器的特性，没必要用 Web 容器去加载服务。
