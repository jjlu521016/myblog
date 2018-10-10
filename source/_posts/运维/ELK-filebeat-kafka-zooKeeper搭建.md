---
title: ELK+filebeat+kafka+zooKeeper搭建（单机版）
toc: true
date: 2018-05-09 15:54:12
tags:
    - ELK
---
本教程中使用的`ELK和filebeat`版本是 `6.2.4`

## 1. elk安装
关于elk的配置参考我之前的一篇文章，不在累述：
elk安装地址：
https://jjlu521016.github.io/2018/05/01/springboot-logback-log4j-elk.html#2-elk%E9%85%8D%E7%BD%AE
<!-- more -->
## 2. kafka及zookeeper安装
参考我之前的文章、把对应的配置改成单机即可：
### 2.1. 安装kafka：
 https://jjlu521016.github.io/2018/01/12/%E8%BF%90%E7%BB%B4/Kafka%E9%9B%86%E7%BE%A4%E6%90%AD%E5%BB%BA%E6%AD%A5%E9%AA%A4.html#more

### 2.2. 安装zookeeper：
https://jjlu521016.github.io/2018/01/01/%E8%BF%90%E7%BB%B4/zookeeper%E9%9B%86%E7%BE%A4%E6%90%AD%E5%BB%BA%E6%AD%A5%E9%AA%A4.html#more

### 2.3 启动zookeeper
我本地zookeeper配置了环境变量
```sh
zkServer.sh start
```
### 2.4 启动kafka
启动kafka
```sh
# bin/kafka-server-start.sh config/server.properties
```
创建topic
```sh
# bin/kafka-topics.sh --create --zookeeper 192.168.188.110:2181 --replication-factor 1 --partitions 1 --topic test 
```
创建生产者
```sh
# bin/kafka-console-producer.sh --broker-list 192.168.188.110:9092 --topic test 
```
创建消费者
```sh
bin/kafka-console-consumer.sh --zookeeper 192.168.188.110:2181 --topic test --from-beginning
```
此时生产者输入的内容可以在消费者输出，证明kafka调用通了。

## 3. 集成logstash与kafka
参考：
https://www.elastic.co/guide/en/logstash/current/plugins-inputs-kafka.html
vi logstash_agent.conf
```conf
input {
  kafka {
    bootstrap_servers => "192.168.188.110:9092"
    #topic id
    topics => "test"
    codec => "json"
  }
}
filter {
    json {
      source => "message"
      remove_field => "message"
    }
}
output {
   elasticsearch{
      hosts => ["192.168.188.110:9200"]
	    index => "kafka-%{+YYYY.MM.dd}"
    }
   stdout { codec => rubydebug }
}

```
启动elk之后，在kafka生产者输入一条信息之后logstash打印出对应的内容，说明集成成功了
。在kibana设置对应的日志。

## 4. 安装filebeat
解压
```sh
tar -zxvf filebeat-6.2.3-linux-x86_64.tar.gz
```
配置filebeat.yml
```yml
filebeat.prospectors:
# 这里面配置实际的日志路径
- type: log
  enabled: true
  paths:
    - /var/log/test1.log
  fields:
    log_topics: test1
- type: log
  enabled: true
  paths:
    - /var/log/test2.log
  fields:
    log_topics: test2
output.kafka:
  enabled: true
  hosts: ["10.112.101.90:9092"]
  topic: '%{[fields][log_topics]}'
```
运行
```sh
./filebeat -e -c filebeat.yml
```
把logstash停止，修改配置文件，添加filebeat配置的topic。重启logstash。打开kibana刷新页面，看到filebeat收集到的日志。

![](https://github.com/jjlu521016/myblog/blob/master/source/image/resource/kibana1.png)

注意：
在logstash中，当input里面有多个kafka输入源时，client_id => "xxx"必须添加且需要不同，否则会报错javax.management.InstanceAlreadyExistsException: kafka.consumer:type=app-info,id=logstash-0。
```conf
input {
  kafka {
    
     group_id => "es1"
   
  }
 
  kafka {
     client_id => "es2"
  }
}
```