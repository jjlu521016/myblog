---
title: dubbo源码解析1-什么是dubbo
tags:
  - dubbo
toc: true
originContent: "# 1. 什么是 dubbo？\n[官方](http://dubbo.apache.org/zh-cn/index.html)给出的定义是：Apache Dubbo (incubating) |ˈdʌbəʊ| 是一款高性能、轻量级的开源Java RPC框架，它提供了三大核心能力：面向接口的远程方法调用，智能容错和负载均衡，以及服务自动注册和发现。\ndubbo产生的背景，详见 [http://dubbo.apache.org/zh-cn/docs/user/preface/background.html](http://dubbo.apache.org/zh-cn/docs/user/preface/background.html)\n# 2. 什么是 RPC？\nRPC（Remote Procedure Call） 详见[Wikipedia](https://en.wikipedia.org/wiki/Remote_procedure_call) 、[IBM](https://www.ibm.com/developerworks/cn/aix/library/au-rpc_programming/index.html)。这里不再累述。\n# 3. dubbo的发展史\n<!-- more -->\n说起dubbo，就不得不提 SOA（Service Oriented Ambiguity），可以看下Martin Fowler在2005年的这篇[文章](https://www.martinfowler.com/bliki/ServiceOrientedAmbiguity.html)\nMartin Fowler认为：SOA意味服务接口，意味流程整合，意味资源再利用，意味着管制，在下面SOA组件图中，服务和服务消费者(客户端)之间存在多个约束，当一个服务显式暴露后，客户端能够通过绑定定位到该服务，相当于两者签订了合同，规定了合同内容和如何实施，具体合同的描述是通过消息方式进行：\n![image.png](/images/2019/02/14/b2d2d290-303d-11e9-9ca1-4912d7f30a5e.png)\n回到正题\n+ 2008年，阿里巴巴开始内部使用 Dubbo。\n+ 2009年初，发布 1.0 版本。\n+ 2010 年初，发布 2.0 版本。\n+ 2011 年 10 月，阿里巴巴宣布开源，版本为 2.0.7。\n+ 2012 年 3 月，发布 2.1.0 版本。这一年dubbo宣称已经每天为2000+个服务提供3,000,000,000+次访问量支持，并被广泛应用于阿里巴巴集团的各成员站点。2012年10月23日Dubbo2.5.3发布后，在Dubbo开源将满一周年之际，阿里基本停止了对Dubbo的主要升级。\n+ 2013 年 3 月，发布 2.4.10 版本。\n+ 2014 年 10 月，发布 2.3.11 版本，之后版本停滞。  \n2013年和2014年更新过2次对Dubbo2.4的维护版本，然后停止了所有维护工作。Dubbo对Spring的支持也停留在了Spring 2.5.6版本上。\n+ 2014 年 10 月，当当网 Fork 了 Dubbo 版本，命名为 Dubbox-2.8.0，并支持 HTTP REST 协议。支持了新版本的Spring，并对外开源了Dubbox\n+ 2017 年 9 月，阿里巴巴重启维护，重点升级所依赖 JDK 及组件版本，发布 2.5.4/5 版本。距离上一个版本2.5.3发布已经接近快5年时间了。在随后的几个月中，阿里Dubbo开发团队以差不多每月一版本的速度开始快速升级迭代，修补了Dubbo老版本多年来存在的诸多bug，并对Spring等组件的支持进行了全面升级。\n+ 2017 年 10 月，发布 2.5.6 版本。\n+ 2017 年 11 月，发布 2.5.7 版本，后期集成 Spring Boot。\n+ 2018年1月8日，Dubbo 2.6.0版本发布，新版本将之前当当网开源的Dubbo分支Dubbox进行了合并，实现了Dubbo版本的统一整合。  \n\ngithub地址：[incubator-dubbo](https://github.com/apache/incubator-dubbo)\n\n# 4. dubbo的架构\n详见[http://dubbo.apache.org/zh-cn/docs/user/preface/architecture.html](http://dubbo.apache.org/zh-cn/docs/user/preface/architecture.html)\n![image.png](/images/2019/02/13/f3e97e90-2f6c-11e9-acfe-2541123bc20a.png)\n\n节点角色说明：  \n\n|节点名称|说明|\n| -----   | ---------------------|\n|Provider | 暴露服务的服务提供方。 |\n|Consumer | 调用远程服务的服务消费方。| \n|Registry | 服务注册与发现的注册中心。| \n|Monitor  | 统计服务的调用次数和调用时间的监控中心。|\n\n调用流程 \n0.服务容器负责启动，加载，运行服务提供者。 \n1.服务提供者在启动时，向注册中心注册自己提供的服务。 \n2.服务消费者在启动时，向注册中心订阅自己所需的服务。 \n3.注册中心返回服务提供者地址列表给消费者，如果有变更，注册中心将基于长连接推送变更数据给消费者。 \n4.服务消费者，从提供者地址列表中，基于软负载均衡算法，选一台提供者进行调用，如果调用失败，再选另一台调用。 \n5.服务消费者和提供者，在内存中累计调用次数和调用时间，定时每分钟发送一次统计数据到监控中心\n\n# 5. Dubbo注册中心\n对于服务提供方，它需要发布服务，而且由于应用系统的复杂性，服务的数量、类型也不断膨胀； \n对于服务消费方，它最关心如何获取到它所需要的服务，而面对复杂的应用系统，需要管理大量的服务调用。 \n而且，对于服务提供方和服务消费方来说，他们还有可能兼具这两种角色，即既需要提供服务，有需要消费服务。\n\n通过将服务统一管理起来，可以有效地优化内部应用对服务发布/使用的流程和管理。服务注册中心可以通过特定协议来完成服务对外的统一。\n\nDubbo提供的注册中心有如下几种类型可供选择：\n\nMulticast注册中心\nZookeeper注册中心\nRedis注册中心\nSimple注册中心\n# 6. 生态系统\n- 脚手架\n  快速生成基于 Spring Boot 的 Dubbo 项目:\n  [Dubbo Initializr](http://start.dubbo.io/)\n- 多语言\n  [java](https://github.com/apache/incubator-dubbo)  \n  [Node.js](https://github.com/dubbo/dubbo2.js)\n  [Python](https://github.com/dubbo/dubbo-client-py)\n  [PHP](https://github.com/dubbo/dubbo-php-framework)\n- API\nDubbo支持通过多种API方式启动:\n  [Spring XML](http://dubbo.apache.org/zh-cn/docs/user/configuration/xml.html)\n  [Spring Annotation](http://dubbo.apache.org/zh-cn/docs/user/configuration/annotation.html)\n  [Plain Java](http://dubbo.apache.org/zh-cn/docs/user/configuration/properties.html)\n  [Spring Boot](https://github.com/apache/incubator-dubbo-spring-boot-project)\n- Registry\nDubbo支持以下注册中心:\nZookeeper\nRedis\nSimpleMulticast\nEtcd3  \n\n- Cluster\nDubbo支持以下容错机制:\nFail over\nFail safe\nFail fast\nFail backForking\nBroadcast  \n\n- Load balance\nDubbo支持以下负载均衡策略:\nRandom\nLeast Active\nRound Robin\nConsistent hash\n- Protocol\nDubbo支持以下协议:\nDubbo\nRMI\nHessian\nHTTP\nWebService\nThrift\nNative Thrift\nMemcached\nRedis\nRest\nJson\nRPC\nXmlRPC\nJmsRpc  \n\n- Transport\nDubbo支持以下网络传输扩展:\n\nNetty3Netty4GrizzlyJettyMinaP2PZookeeper\nSerialization\nDubbo支持以下序列化机制:\n\nHessian2JavaJSONFstKryoNative HessianAvro\n\n# 7.Dubbo优缺点\n优点：\n透明化的远程方法调用 \n- 像调用本地方法一样调用远程方法；只需简单配置，没有任何API侵入。\n软负载均衡及容错机制 \n可在内网替代nginx lvs等硬件负载均衡器。\n服务注册中心自动注册 & 配置管理 \n- 不需要写死服务提供者地址，注册中心基于接口名自动查询提供者ip。 \n使用类似zookeeper等分布式协调服务作为服务注册中心，可以将绝大部分项目配置移入zookeeper集群。\n服务接口监控与治理 \n- Dubbo-admin与Dubbo-monitor提供了完善的服务接口管理与监控功能，针对不同应用的不同接口，可以进行 多版本，多协议，多注册中心管理。\n\n缺点：\n- Registry 严重依赖第三方组件（zookeeper 或者 redis），当这些组件出现问题时，服务调用很快就会中断。\n- Dubbo2.x 只支持 RPC 调用。使得服务提供方（抽象接口）与调用方在代码上产生了强依赖，服务提供者需要不断将包含抽象接口的 jar 包打包出来供消费者使用。一旦打包出现问题，就会导致服务调用出错，并且以后发布部署会成很大问题（太强的依赖关系）。\n- Dubbo 只是实现了服务治理，其他微服务框架并未包含，如果需要使用，需要结合第三方框架实现（比如分布式配置用淘宝的 Diamond、服务跟踪用京东的 Hydra，但使用相对麻烦些），开发成本较高，且风险较大。\n- 主要是国内公司使用，但阿里内部使用 HSF，相对于 Spring Cloud，企业应用会差一些。\n\n\n# 8. Dubbo和Spring Cloud对比\nspring Cloud架构\n\nSpring Cloud总体架构如下图\n![image.png](/images/2019/02/14/65488610-3041-11e9-9ca1-4912d7f30a5e.png)  \n\n|节点名称\t|   说明   |\n| ------------- | --------- |\n|Service Provider | 暴露服务的提供方。|\n|Service Consumer | 调用远程服务的服务消费方。|\n|EureKa Server     | 服务注册中心和服务发现中心。|\nspring Cloud组件图\n![image.png](/images/2019/02/14/332617a0-3042-11e9-9ca1-4912d7f30a5e.png)  \n\n\n|名称|Dubbo|Spring Cloud|\n|-|-|-|\n|服务注册中心|Zookeeper/Redis|Eureka|\n|服务调用方式|\tRPC\t    |REST API |\n|断路器\t    |  不完善\t    |Hystrix  |\n|分布式配置  |\t(Nacos)\t    |Spring Cloud Config(Nacos)|\n|服务跟踪    |\t无\t    |Spring Cloud Sleuth|\n|消息总线    |\t无\t    |Spring Cloud Bus|\n|数据流\t    | 无\t    |Spring Cloud Stream|\n|批量任务    |\t无\t    |Spring Cloud Task|\n.....\n\n大致可以理解为什么说Dubbo只是类似Netflix的一个子集了吧。\n需要申明一点，Dubbo对于上表中总结为“无”的组件不代表不能实现，而只是Dubbo框架自身不提供，需要另外整合以实现对应的功能"
categories:
  - dubbo
date: 2019-02-13 16:52:55
---

# 1. 什么是 dubbo？
[官方](http://dubbo.apache.org/zh-cn/index.html)给出的定义是：Apache Dubbo (incubating) |ˈdʌbəʊ| 是一款高性能、轻量级的开源Java RPC框架，它提供了三大核心能力：面向接口的远程方法调用，智能容错和负载均衡，以及服务自动注册和发现。
dubbo产生的背景，详见 [http://dubbo.apache.org/zh-cn/docs/user/preface/background.html](http://dubbo.apache.org/zh-cn/docs/user/preface/background.html)
# 2. 什么是 RPC？
RPC（Remote Procedure Call） 详见[Wikipedia](https://en.wikipedia.org/wiki/Remote_procedure_call) 、[IBM](https://www.ibm.com/developerworks/cn/aix/library/au-rpc_programming/index.html)。这里不再累述。
# 3. dubbo的发展史
<!-- more -->
说起dubbo，就不得不提 SOA（Service Oriented Ambiguity），可以看下Martin Fowler在2005年的这篇[文章](https://www.martinfowler.com/bliki/ServiceOrientedAmbiguity.html)
Martin Fowler认为：SOA意味服务接口，意味流程整合，意味资源再利用，意味着管制，在下面SOA组件图中，服务和服务消费者(客户端)之间存在多个约束，当一个服务显式暴露后，客户端能够通过绑定定位到该服务，相当于两者签订了合同，规定了合同内容和如何实施，具体合同的描述是通过消息方式进行：
![image.png](/images/2019/02/14/b2d2d290-303d-11e9-9ca1-4912d7f30a5e.png)
回到正题
+ 2008年，阿里巴巴开始内部使用 Dubbo。
+ 2009年初，发布 1.0 版本。
+ 2010 年初，发布 2.0 版本。
+ 2011 年 10 月，阿里巴巴宣布开源，版本为 2.0.7。
+ 2012 年 3 月，发布 2.1.0 版本。这一年dubbo宣称已经每天为2000+个服务提供3,000,000,000+次访问量支持，并被广泛应用于阿里巴巴集团的各成员站点。2012年10月23日Dubbo2.5.3发布后，在Dubbo开源将满一周年之际，阿里基本停止了对Dubbo的主要升级。
+ 2013 年 3 月，发布 2.4.10 版本。
+ 2014 年 10 月，发布 2.3.11 版本，之后版本停滞。  
2013年和2014年更新过2次对Dubbo2.4的维护版本，然后停止了所有维护工作。Dubbo对Spring的支持也停留在了Spring 2.5.6版本上。
+ 2014 年 10 月，当当网 Fork 了 Dubbo 版本，命名为 Dubbox-2.8.0，并支持 HTTP REST 协议。支持了新版本的Spring，并对外开源了Dubbox
+ 2017 年 9 月，阿里巴巴重启维护，重点升级所依赖 JDK 及组件版本，发布 2.5.4/5 版本。距离上一个版本2.5.3发布已经接近快5年时间了。在随后的几个月中，阿里Dubbo开发团队以差不多每月一版本的速度开始快速升级迭代，修补了Dubbo老版本多年来存在的诸多bug，并对Spring等组件的支持进行了全面升级。
+ 2017 年 10 月，发布 2.5.6 版本。
+ 2017 年 11 月，发布 2.5.7 版本，后期集成 Spring Boot。
+ 2018年1月8日，Dubbo 2.6.0版本发布，新版本将之前当当网开源的Dubbo分支Dubbox进行了合并，实现了Dubbo版本的统一整合。  

github地址：[incubator-dubbo](https://github.com/apache/incubator-dubbo)

# 4. dubbo的架构
详见[http://dubbo.apache.org/zh-cn/docs/user/preface/architecture.html](http://dubbo.apache.org/zh-cn/docs/user/preface/architecture.html)
![image.png](/images/2019/02/13/f3e97e90-2f6c-11e9-acfe-2541123bc20a.png)

节点角色说明：  

|节点名称|说明|
| -----   | ---------------------|
|Provider | 暴露服务的服务提供方。 |
|Consumer | 调用远程服务的服务消费方。| 
|Registry | 服务注册与发现的注册中心。| 
|Monitor  | 统计服务的调用次数和调用时间的监控中心。|

调用流程 
0.服务容器负责启动，加载，运行服务提供者。 
1.服务提供者在启动时，向注册中心注册自己提供的服务。 
2.服务消费者在启动时，向注册中心订阅自己所需的服务。 
3.注册中心返回服务提供者地址列表给消费者，如果有变更，注册中心将基于长连接推送变更数据给消费者。 
4.服务消费者，从提供者地址列表中，基于软负载均衡算法，选一台提供者进行调用，如果调用失败，再选另一台调用。 
5.服务消费者和提供者，在内存中累计调用次数和调用时间，定时每分钟发送一次统计数据到监控中心

# 5. 生态系统
- 脚手架
  快速生成基于 Spring Boot 的 Dubbo 项目:
  [Dubbo Initializr](http://start.dubbo.io/)
- 多语言
  [java](https://github.com/apache/incubator-dubbo)  
  [Node.js](https://github.com/dubbo/dubbo2.js)
  [Python](https://github.com/dubbo/dubbo-client-py)
  [PHP](https://github.com/dubbo/dubbo-php-framework)
- API
Dubbo支持通过多种API方式启动:
  [Spring XML](http://dubbo.apache.org/zh-cn/docs/user/configuration/xml.html)
  [Spring Annotation](http://dubbo.apache.org/zh-cn/docs/user/configuration/annotation.html)
  [Plain Java](http://dubbo.apache.org/zh-cn/docs/user/configuration/properties.html)
  [Spring Boot](https://github.com/apache/incubator-dubbo-spring-boot-project)
- Registry
对于服务提供方，它需要发布服务，而且由于应用系统的复杂性，服务的数量、类型也不断膨胀；
对于服务消费方，它最关心如何获取到它所需要的服务，而面对复杂的应用系统，需要管理大量的服务调用。
而且，对于服务提供方和服务消费方来说，他们还有可能兼具这两种角色，即既需要提供服务，有需要消费服务。
通过将服务统一管理起来，可以有效地优化内部应用对服务发布/使用的流程和管理。服务注册中心可以通过特定协议来完成服务对外的统一。
Dubbo支持以下注册中心:
[Zookeeper](http://dubbo.apache.org/zh-cn/docs/user/references/registry/zookeeper.html)
[Redis](http://dubbo.apache.org/zh-cn/docs/user/references/registry/redis.html)
[Simple](http://dubbo.apache.org/zh-cn/docs/user/references/registry/simple.html)
[Multicast](http://dubbo.apache.org/zh-cn/docs/user/references/registry/multicast.html)
[Etcd3](https://github.com/dubbo/dubbo-registry-etcd) 

- Cluster
Dubbo支持以下容错机制:
[Fail over](http://dubbo.apache.org/zh-cn/docs/user/demos/fault-tolerent-strategy.html)
[Fail safe](http://dubbo.apache.org/zh-cn/docs/user/demos/fault-tolerent-strategy.html)
[Fail fast](http://dubbo.apache.org/zh-cn/docs/user/demos/fault-tolerent-strategy.html)
[Fail back](http://dubbo.apache.org/zh-cn/docs/user/demos/fault-tolerent-strategy.html)
[Forking](http://dubbo.apache.org/zh-cn/docs/user/demos/fault-tolerent-strategy.html)
[Broadcast](http://dubbo.apache.org/zh-cn/docs/user/demos/fault-tolerent-strategy.html) 

- Load balance
Dubbo支持以下负载均衡策略:
[Random](http://dubbo.apache.org/zh-cn/docs/user/demos/loadbalance.html)
[Least Active](http://dubbo.apache.org/zh-cn/docs/user/demos/loadbalance.html)
[Round Robin](http://dubbo.apache.org/zh-cn/docs/user/demos/loadbalance.html)
[Consistent hash](http://dubbo.apache.org/zh-cn/docs/user/demos/loadbalance.html)
- Protocol
Dubbo支持以下协议:
[Dubbo](http://dubbo.apache.org/zh-cn/docs/user/references/protocol/dubbo.html)
[RMI](http://dubbo.apache.org/zh-cn/docs/user/references/protocol/rmi.html)
[Hessian](http://dubbo.apache.org/zh-cn/docs/user/references/protocol/hessian.html)
[HTTP](http://dubbo.apache.org/zh-cn/docs/user/references/protocol/http.html)
[WebService](http://dubbo.apache.org/zh-cn/docs/user/references/protocol/webservice.html)
[Thrift](http://dubbo.apache.org/zh-cn/docs/user/references/protocol/thrift.html)
[Native Thrift](https://github.com/dubbo/dubbo-rpc-native-thrift)
[Memcached](http://dubbo.apache.org/zh-cn/docs/user/references/protocol/memcached.html)
[Redis](http://dubbo.apache.org/zh-cn/docs/user/references/protocol/redis.html)
[Rest](http://dubbo.apache.org/zh-cn/docs/user/references/protocol/rest.html)
[JsonRPC](https://github.com/apache/incubator-dubbo-rpc-jsonrpc)
[XmlRPC](https://github.com/dubbo/incubator-dubbo-rpc-xmlrpc)
[JmsRpc](https://github.com/dubbo/incubator-dubbo-rpc-jms)  

- Transport
Dubbo支持以下网络传输扩展:
[Netty3](http://dubbo.apache.org/zh-cn/community/index.html)
[Netty4](http://dubbo.apache.org/zh-cn/docs/user/demos/netty4.html)
[Grizzly](http://dubbo.apache.org/zh-cn/community/index.html)
[Jetty](http://dubbo.apache.org/zh-cn/community/index.html)
[Mina](http://dubbo.apache.org/zh-cn/community/index.html)
[P2P](http://dubbo.apache.org/zh-cn/community/index.html)
[Zookeeper](http://dubbo.apache.org/zh-cn/community/index.html)  

- Serialization
Dubbo支持以下序列化机制:
[Hessian2](http://dubbo.apache.org/zh-cn/community/index.html)
[Java](http://dubbo.apache.org/zh-cn/community/index.html)
[JSON](http://dubbo.apache.org/zh-cn/community/index.html)
[Fst](http://dubbo.apache.org/zh-cn/community/index.html)
[Kryo](http://dubbo.apache.org/zh-cn/community/index.html)
[Native Hessian](https://github.com/dubbo/dubbo-serialization-native-hessian)
[Avro](https://github.com/dubbo/dubbo-serialization-avro)

# 6. Dubbo和Spring Cloud对比
spring Cloud架构

Spring Cloud总体架构如下图
![image.png](/images/2019/02/14/65488610-3041-11e9-9ca1-4912d7f30a5e.png)  

|节点名称	|   说明   |
| ------------- | --------- |
|Service Provider | 暴露服务的提供方。|
|Service Consumer | 调用远程服务的服务消费方。|
|EureKa Server     | 服务注册中心和服务发现中心。|
spring Cloud组件图
![image.png](/images/2019/02/14/332617a0-3042-11e9-9ca1-4912d7f30a5e.png)  


|名称|Dubbo|Spring Cloud|
|-|-|-|
|服务注册中心|Zookeeper/Redis|Eureka|
|服务调用方式|	RPC	    |REST API |
|断路器	    |  不完善	    |Hystrix  |
|分布式配置  |	(Nacos)	    |Spring Cloud Config(Nacos)|
|服务跟踪    |	无	    |Spring Cloud Sleuth|
|消息总线    |	无	    |Spring Cloud Bus|
|数据流	    | 无	    |Spring Cloud Stream|
|批量任务    |	无	    |Spring Cloud Task|
.....

大致可以理解为什么说Dubbo只是类似Netflix的一个子集了吧。
需要申明一点，Dubbo对于上表中总结为“无”的组件不代表不能实现，而只是Dubbo框架自身不提供，需要另外整合以实现对应的功能