---
title: Logstash配置文件简述
tags:
  - ELK
toc: true
originContent: "# 1. logstash conf文件结构及语法\n## 1.1 conf文件结构\n官方说明请参考  \nhttps://www.elastic.co/guide/en/logstash/5.4/configuration-file-structure.html\n```conf\ninput {\n\n}\nfilter {\n\n}\noutput {\n\n}\n```\n## 1.2 配置文件语法\n### 1.2.1 基本语法\n\n```conf\n数据类型\nboolen： 布尔 a => true\nBytes： 字节 a => “10MiB”\nStrings：字符串 a => “hello world”\nNumber： 数值 a => 1024\nArray： 数组 match => [“datatime”,“UNIX”,“ISO8601”]\nHash： 哈希 options => { key1 => “value1”,key2 => “value2” }\n编码解码： codec: codec => “json”\n密码型： my_passwd => “password”\n路径： my_path => “/tmp/logstash”\n注释： #\n```\n### 1.2.2 条件判断\n```conf\n==,!= ,< ,> ,<= ,>=\n=~\nin,not in\nand ,or , nand, xor\n(), !()\nif expression {\n} else if expression {\n…\n} else {\n…\n}\n```\n### 1.2.3 字段引用\n```conf\n%{[response][status]}\n%{[@metadata][kafka][topic]}\n```\n--------------------- \n\n# 2. 各部分配置详解\n## 2.1 input\n官方input插件列表\nhttps://www.elastic.co/guide/en/logstash/current/input-plugins.html\n具体配置参考官方说明，这部分比较简单，我们就用kafka插件举例\n```conf\ninput {\n  # https://www.elastic.co/guide/en/logstash/6.2/plugins-inputs-kafka.html\n  kafka {\n     #kafka Server\n     bootstrap_servers => \"xxx:9092\"\n    #topic id\n    topics => [\"xxx_tomcat\",\"xx2\"]\n    # 这里group_id (默认为logstash)需要解释一下，在Kafka中，相同group的Consumer可以同时消费一个topic，不同group的Consumer工作则互不干扰。\n    # 补充: 在同一个topic中的同一个partition同时只能由一个Consumer消费，当同一个topic同时需要有多个Consumer消费时，则可以创建更多的partition。\n    group_id => \"xxx\"\n    # 当input里面有多个kafka输入源时，client_id => \"es*\"必须添加且需要不同，\n    # 否则会报错javax.management.InstanceAlreadyExistsException: kafka.consumer:type=app-info,id=logstash-0。\n    client_id => \"xxx\"\n    #https://blog.csdn.net/nyyjs/article/details/72771905\n    consumer_threads => 2\n    decorate_events => true\n    codec => json {\n            charset => \"UTF-8\"\n      }\n    auto_offset_reset => \"latest\"\n  }\n}\n```\n## 2.2 filter\n官方filter插件列表\nhttps://www.elastic.co/guide/en/logstash/current/filter-plugins.html\n这部分是logstash最复杂的一个地方，也是logstash解析日志最核心的地方\n一般我们常用的插件有  \n\n- date 日期相关\n- geoip 解析地理位置相关\n- mutate 对指定字段的增删改\n- grok 将message中的数据解析成es中存储的字段  \n\n其中grok和mutate是用的最多的地方，这块大家可以多看下官方的文档。\n下面用一个filebeat -> kafka的数据来演示用法\n其中grok的官方正则参考地址如下：\nhttps://github.com/logstash-plugins/logstash-patterns-core/blob/master/patterns/grok-patterns\n```conf\nfilter {\n      #xxx_tomcat是topic名字\n\t if \"xxx_tomcat\" ==  [@metadata][kafka][topic] {\n      grok{\n\t      #指定自定义正则文件地址，如果使用官方的正则，不需要配置这个\n          patterns_dir => \"/data/elk/logstash-6.2.4/patterns\"\n          match => { \"message\" => \"%{TOMCAT_xxx}\"}\n      }\n      date {\n          match => [\"logDate\", \"yyyy-MM-dd;HH:mm:ss.SSS\"]\n      }\n      mutate {\n        remove_field => [\"logDate\"]\n      }\n     }\n\t #修改field\n     mutate {\n        rename => {\"[beat][name]\"=>\"serverName\"}\n     }\n    # 删除无用字段 这些字段kafka和filebeat\n    # 不能移除 type字段，否则会导致不能自动生成索引\n    mutate {\n    remove_field => [\"_score\",\"_id\", \"@version\" , \"_type\" , \"offset\" ,  \n                      \"version\",\"id\" , \"score\", \"tags\", \"source\",\"sort\",\n                      \"prospector\",\"[beat][version]\",\"[beat][hostname]\",\"_score\",\"fields\" ]\n    # remove_field => \"type\"\n\n  }\n}\n```\n## 2.2 output\n官方filter插件列表\nhttps://www.elastic.co/guide/en/logstash/current/output-plugins.html\n这块也是比较简单的，按照插件的解释就可以配置成功，下面我们以ES为例来看下\n```conf\noutput {\n\n  if \"xxx_tomcat\" ==  [@metadata][kafka][topic]  {\n      elasticsearch{\n        hosts => [\"xxx:9200\"]\n        index => \"%{[@metadata][kafka][topic]}-%{+YYYY.MM.dd}\"\n\t\t#如果配置了elasticsearch认证的话需要配置下面的user和password，否则不需要。\n        user => \"xxx\"\n        password => \"xxxx\"\n      }\n  }\n  #输出日志到控制台，会输出具体的es数据内容，调试完成后建议去掉。\n  stdout { codec => rubydebug }\n}\n```\n\n# 3. 启动加载配置文件\n启动的时候可以指定文件或者文件目录下的所有 `.conf`文件。\n# 3.1 加载具体配置文件：\n\n```sh\n./bin/logstash -f config/test-kafka.conf\n```\n# 3.2 加载配置文件目录：\n假设配置文件都在 config/config.d\n```sh\n./bin/logstash -f config/config.d\n```\n# 4. 总结\nlogstash配置文件的难点就是grok这块，建议在使用的时候多看下官方相关的文档。"
categories:
  - 运维
date: 2018-12-07 12:41:32
---

# 1. logstash conf文件结构及语法
## 1.1 conf文件结构
官方说明请参考  
https://www.elastic.co/guide/en/logstash/5.4/configuration-file-structure.html
```conf
input {

}
filter {

}
output {

}
```
## 1.2 配置文件语法
### 1.2.1 基本语法

```conf
数据类型
boolen： 布尔 a => true
Bytes： 字节 a => “10MiB”
Strings：字符串 a => “hello world”
Number： 数值 a => 1024
Array： 数组 match => [“datatime”,“UNIX”,“ISO8601”]
Hash： 哈希 options => { key1 => “value1”,key2 => “value2” }
编码解码： codec: codec => “json”
密码型： my_passwd => “password”
路径： my_path => “/tmp/logstash”
注释： #
```
### 1.2.2 条件判断
```conf
==,!= ,< ,> ,<= ,>=
=~
in,not in
and ,or , nand, xor
(), !()
if expression {
} else if expression {
…
} else {
…
}
```
### 1.2.3 字段引用
```conf
%{[response][status]}
%{[@metadata][kafka][topic]}
```
--------------------- 

# 2. 各部分配置详解
## 2.1 input
官方input插件列表
https://www.elastic.co/guide/en/logstash/current/input-plugins.html
具体配置参考官方说明，这部分比较简单，我们就用kafka插件举例
```conf
input {
  # https://www.elastic.co/guide/en/logstash/6.2/plugins-inputs-kafka.html
  kafka {
     #kafka Server
     bootstrap_servers => "xxx:9092"
    #topic id
    topics => ["xxx_tomcat","xx2"]
    # 这里group_id (默认为logstash)需要解释一下，在Kafka中，相同group的Consumer可以同时消费一个topic，不同group的Consumer工作则互不干扰。
    # 补充: 在同一个topic中的同一个partition同时只能由一个Consumer消费，当同一个topic同时需要有多个Consumer消费时，则可以创建更多的partition。
    group_id => "xxx"
    # 当input里面有多个kafka输入源时，client_id => "es*"必须添加且需要不同，
    # 否则会报错javax.management.InstanceAlreadyExistsException: kafka.consumer:type=app-info,id=logstash-0。
    client_id => "xxx"
    #https://blog.csdn.net/nyyjs/article/details/72771905
    consumer_threads => 2
    decorate_events => true
    codec => json {
            charset => "UTF-8"
      }
    auto_offset_reset => "latest"
  }
}
```
## 2.2 filter
官方filter插件列表
https://www.elastic.co/guide/en/logstash/current/filter-plugins.html
这部分是logstash最复杂的一个地方，也是logstash解析日志最核心的地方
一般我们常用的插件有  

- date 日期相关
- geoip 解析地理位置相关
- mutate 对指定字段的增删改
- grok 将message中的数据解析成es中存储的字段  

其中grok和mutate是用的最多的地方，这块大家可以多看下官方的文档。
下面用一个filebeat -> kafka的数据来演示用法
其中grok的官方正则参考地址如下：
https://github.com/logstash-plugins/logstash-patterns-core/blob/master/patterns/grok-patterns
```conf
filter {
      #xxx_tomcat是topic名字
	 if "xxx_tomcat" ==  [@metadata][kafka][topic] {
      grok{
	      #指定自定义正则文件地址，如果使用官方的正则，不需要配置这个
          patterns_dir => "/data/elk/logstash-6.2.4/patterns"
          match => { "message" => "%{TOMCAT_xxx}"}
      }
      date {
          match => ["logDate", "yyyy-MM-dd;HH:mm:ss.SSS"]
      }
      mutate {
        remove_field => ["logDate"]
      }
     }
	 #修改field
     mutate {
        rename => {"[beat][name]"=>"serverName"}
     }
    # 删除无用字段 这些字段kafka和filebeat
    # 不能移除 type字段，否则会导致不能自动生成索引
    mutate {
    remove_field => ["_score","_id", "@version" , "_type" , "offset" ,  
                      "version","id" , "score", "tags", "source","sort",
                      "prospector","[beat][version]","[beat][hostname]","_score","fields" ]
    # remove_field => "type"

  }
}
```
## 2.2 output
官方filter插件列表
https://www.elastic.co/guide/en/logstash/current/output-plugins.html
这块也是比较简单的，按照插件的解释就可以配置成功，下面我们以ES为例来看下
```conf
output {

  if "xxx_tomcat" ==  [@metadata][kafka][topic]  {
      elasticsearch{
        hosts => ["xxx:9200"]
        index => "%{[@metadata][kafka][topic]}-%{+YYYY.MM.dd}"
		#如果配置了elasticsearch认证的话需要配置下面的user和password，否则不需要。
        user => "xxx"
        password => "xxxx"
      }
  }
  #输出日志到控制台，会输出具体的es数据内容，调试完成后建议去掉。
  stdout { codec => rubydebug }
}
```

# 3. 启动加载配置文件
启动的时候可以指定文件或者文件目录下的所有 `.conf`文件。
# 3.1 加载具体配置文件：

```sh
./bin/logstash -f config/test-kafka.conf
```
# 3.2 加载配置文件目录：
假设配置文件都在 config/config.d
```sh
./bin/logstash -f config/config.d
```
# 4. 总结
logstash配置文件的难点就是grok这块，建议在使用的时候多看下官方相关的文档。