---
title: springboot logback(log4j) elk 非集群
toc: true
date: 2018-05-1 18:06:17
tags:
  - elk
---
好长时间没有写过blog了。抽时间把很久之前集成的一个简易的elk升级了下
本教程使用的软件如下：
springboot 2.*
jdk8
elk 6.2.4(elasticsearch logstash kibana)
<!-- more -->
### 1. springboot集成logback+logstash
### 1.1 pom加入依赖
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-logging</artifactId>
    <version>2.0.0.RELEASE</version>
</dependency>
<!-- Logstash encoder -->
<dependency>
    <groupId>net.logstash.logback</groupId>
    <artifactId>logstash-logback-encoder</artifactId>
    <version>4.9</version>
</dependency>
<dependency>
    <groupId>net.logstash.log4j</groupId>
    <artifactId>jsonevent-layout</artifactId>
    <version>1.7</version>
</dependency>
```
### 1.2 在application.yml中加载logback的配置文件
```yml
logging:
  config: classpath:logback.xml
  path: logs
```
### 1.3 在logback.xml中配置logstash
```xml
<appender name="logstash" class="net.logstash.logback.appender.LogstashTcpSocketAppender">
    <param name="Encoding" value="UTF-8"/>
    <destination>192.168.188.110</destination>
    <port>4560</port>
    <!-- encoder is required -->
    <encoder class="net.logstash.logback.encoder.LogstashEncoder" />
</appender>

<root level="INFO">
        <appender-ref ref="logstash" />
</root>
```

### 2.elk配置
#### 2.1 es配置
解压文件 
```sh
tar zxvf elasticsearch-6.2.4.tar.gz 
```
由于es不能用root账户启动，所以需要添加一个非root账户
```sh
useradd es
```
修改es文件夹的权限
```sh
chown -R es:es elasticsearch-6.2.4
```
修改配置文件
```sh
vi /opt/elk/elasticsearch-6.2.4/config/elasticsearch.yml
```
修改`elasticsearch.yml`的内容如下
```yml
#端口
http.port: 9200
#ip
network.host: 192.168.188.111
#data路径
path.data: /opt/elk/elasticsearch-6.2.4/data
#logs路径
path.logs: /opt/elk/elasticsearch-6.2.4/logs
```
创建data和logs文件夹
```sh
mkdir data logs
```
启动es(为了方便观察日志没有后台启动)
```
./bin/elasticsearch
```
#### 2.2 logstash 配置

解压文件 
```sh
tar zxvf logstash-6.2.4.tar.gz 
```
进入config文件夹新建log4j_es.conf文件并编写内容
```sh
vi log4j_es.conf
```
log4j_es.conf内容
```conf
input {
    # https://www.elastic.co/guide/en/logstash/current/plugins-inputs-log4j.html
    tcp {  
    mode => "server"  
    host => "192.168.188.110"  
    port => 4560  
    codec => json_lines  
  }
}
output {
   elasticsearch{
        hosts => ["192.168.188.110:9200"]
        index => "log4j-%{+YYYY.MM.dd}"
        document_type => "log4j_type"
    }
   stdout { codec => rubydebug }
}

```
启动logstash (为了方便观察日志没有后台启动)
```sh
./bin/logstash -f config/log4j_es.conf 
```
#### 2.3 kibana 配置
解压
```sh
tar zxvf kibana-6.2.4-linux-x86_64.tar.gz
```
修改配置文件`kibana.yml`
```yml
server.port: 5601
# To allow connections from remote users, set this parameter to a non-loopback address.
server.host: "192.168.188.110"

# The URL of the Elasticsearch instance to use for all your queries.
elasticsearch.url: "http://192.168.188.110:9200"
```
启动kibana
```
./bin/kibana
```
### 3.kibana界面设置
进入kibana界面
点击 Management 点击Index Patterns
在 Create index pattern 的文本框输入索引名称，因为我在logstash中设置索引为 `log4j-%{+YYYY.MM.dd}`,所以我们填写 `log4j-*` 点击下一步设置直到完成。
点击`Discovery`可以看到我们的日志了。

接下来，将elk进行升级 使用elk+filebeat+kafka,敬请期待。