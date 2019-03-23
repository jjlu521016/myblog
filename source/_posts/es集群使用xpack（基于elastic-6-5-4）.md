---
title: ' es集群使用xpack（基于elastic 6.5.4）'
tags:
  - elk
  - 集群
toc: true
originContent: ''
categories:
  - 运维
date: 2019-03-23 16:02:30
---


> 集群搭建过程略
## 1. 启用trial license
> 如果已经有正式license可以忽略这个步骤  

执行命令激活xpack
```sh
curl -H "Content-Type:application/json" -XPOST  http://xxx:9200/_xpack/license/start_trial?acknowledge=true
```
## 1.2. es开启xpack
```yml

xpack.security.enabled: true
```
> 设置密码，在master设置，node节点可以同步该用户名/密码

到此为止完成xpack集群,目前无SSL。

# 2. 使用SSL
## 2.1 master节点生成证书
```sh
./bin/elasticsearch-certutil ca
```
保存`elastic-stack-ca.p12`路径并输入密码（123456）
```sh
./bin/elasticsearch-certutil cert --ca elastic-stack-ca.p12
```
保存`elastic-certificates.p12`路径并输入密码（123456）
将上面生成的两个文件拷贝到elastic的config目录下
比如我设置的是在config/certs下面


## 2.2 证书拷贝至所有elasticsearch节点
- 所有elasticsearch节点启用SSL
```sh
scp -r config/certs/ elk@node2:/opt/elasticsearch-6.5.4/config/
```

## 2.3 elasticsearch.yml中增加配置
```yml
xpack.security.enabled: true
xpack.security.transport.ssl.enabled: true
xpack.security.transport.ssl.verification_mode: certificate
xpack.security.transport.ssl.keystore.path: certs/elastic-certificates.p12
xpack.security.transport.ssl.truststore.path: certs/elastic-certificates.p12
```  

## 2.4 所有elasticsearch节点将密码添加至elasticsearch-keystore
```sh

]$ bin/elasticsearch-keystore add xpack.security.transport.ssl.keystore.secure_password
]$ bin/elasticsearch-keystore add xpack.security.transport.ssl.truststore.secure_password
```