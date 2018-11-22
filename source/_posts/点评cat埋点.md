---
title: 点评cat埋点
toc: true
date: 2018-10-23 09:45:36
tags: 
    - cat
    - 分布式链路跟踪
---
# 1. tracing 演进时间线
<img src='/image/tracing/tracingTime.jpg'>
<!-- more -->

从上图可以看出`CAT（Central Application Tracking）`出现的还是比较早的。CAT整个产品研发是从2011年底开始的，当时正是大众点评从.NET迁移到Java的核心起步阶段。当初大众点评已经有核心的基础中间件、RPC组件Pigeon、统一配置组件Lion。整体Java迁移已经在服务化的路上。随着服务化的深入，整体Java在线上部署规模逐渐变多，同时，暴露的问题也越来越多。
CAT从开发至今，一直秉承着简单的架构就是最好的架构原则，主要分为三个模块：  
+ cat-client（官方表示后期会废弃该模块） 提供给业务以及中间层埋点的底层SDK。
+ cat-consumer 用于实时分析从客户端提供的数据。
+ cat-home 作为用户给用户提供展示的控制端。
详细信息参考: 
https://zhuanlan.zhihu.com/p/23351994  
https://github.com/dianping/cat/blob/master/cat-doc/posts/ch5-design/server.md  

本文不在累述cat的搭建过程，目前官方github的说明文档（3.0版本）已经修复之前若干错误问题。本文只是简单聊下我在集成cat遇到的坑

# 2.集成cat遇到的坑：  

+ 1. 全局`client.xml`目录和log目录默认不可更换
这个问题可以说很严重也可以说可以忽略不计（如果服务器对每个目录都有权限限制，就是一个严重的问题）
经过研究源码发现，其实要想修改这两个也是可以的只是相对麻烦
    + (1) 修改client.xml配置路径为可配置相关代码（基于3.0版本代码，之前版本更麻烦）：
    ```java
     @Value("${cat.client.path}")
    private String CAT_CLIENT_PATH = "";

    /**
        * 初始化cat相关配置默认是 /data/appdatas/cat
        */
    private void loadConfigFile() {
        String catClientXml = CAT_CLIENT_PATH;
        System.setProperty("catXmlPath",catClientXml);
        File file = new File(catClientXml);
        if (!file.exists()) {
            file = new File(Cat.getCatHome() + "client.xml");
        }
        Cat.initialize(file);
    }

    ```
    + (2) 修改打印日志目录
    主要是修改 plexus相关的配置  
    plexus相关的说明可以参考 https://github.com/dianping/cat/blob/v3.0.0/%E7%BE%8E%E5%9B%A2%E7%82%B9%E8%AF%84cat%E5%89%96%E6%9E%90.docx  
    在项目资源路径下新建`META-INF/plexus`文件夹,之后创建`plexus.xml`，其他方法不能修改日志路径
    ```xml
    <plexus>
        <components>
            <component>
                <role>org.codehaus.plexus.logging.LoggerManager</role>
                <implementation>org.unidal.lookup.logger.TimedConsoleLoggerManager</implementation>
                <configuration>
                    <dateFormat>MM-dd HH:mm:ss.SSS</dateFormat>
                    <showClass>true</showClass>
                    <logFilePattern>cat_{0,date,yyyyMMdd}.log</logFilePattern>
                    <baseDirRef>CAT_HOME</baseDirRef>
                    <defaultBaseDir>日志的绝对路径</defaultBaseDir>
                </configuration>
            </component>
        </components>
    </plexus>
    ```
+ 2. 资源少并且杂乱
目前官方3.0版本相关的文档已经可以正常使用了！！！！之前的惨不忍睹！！！
官方文档没有过多的说明埋点的细节，也没有一个统一的入口，只是大致说了 Transaction的用法，官方给的几个集成组件又太少，只能满足一些传统技术的接入，比如springCloud并没有相关的介绍
到网上搜索相关的资源几乎都是一样的。

+ 3. 展示消息树不够直观
展示链路消息的时候，默认情况下要么只能展示详细数据要么只能展示跟踪路径，没有把两者结合，如果基于官方开发相关页面又浪费时间

+ 4. 不能在日志中展示追踪信息
由于cat是基于消息树，没有traceId这个概念，如果想要在日志中能看到对应的数据。我们做链路追踪不仅仅只是关心链路情况。当链路中有异常情况，我们需要根据traceId很快的找到对应的问题。

未完待续.....