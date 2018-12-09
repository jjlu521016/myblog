---
title: Jedis源码阅读笔记(v3.0.0)
toc: true
date: 2018-12-08 21:07:56
tags:
---
jedis的官方api如下:
http://xetorthio.github.io/jedis/

# 1.调用Jedis
我们在调用jedis的时候，一般是如下方式调用Jedis
```java
JedisPool jedisPool = new JedisPool(jedisPoolConfig,host,port,timeout,null);
```
# 2. Jedis源码解析
## 2.1 jedis相关的uml
<image src="/image/jedis/jedis.png">
<!-- more -->  

```java
public class Jedis extends BinaryJedis implements JedisCommands, MultiKeyCommands,
    AdvancedJedisCommands, ScriptingCommands, BasicCommands, ClusterCommands, SentinelCommands, ModuleCommands 
```    
由上图及代码我们可以看出，Jedis类继承了Binary并且实现了 xxxCommands接口
### 2.1.1 BinaryJedis
Jedis在初始化的是会调用BinaryJedis类：
<image src="/image/jedis/binaryJedis.png">

```java
  public BinaryJedis(final String host, final int port, final boolean ssl,
      final SSLSocketFactory sslSocketFactory, final SSLParameters sslParameters,
      final HostnameVerifier hostnameVerifier) {
    client = new Client(host, port, ssl, sslSocketFactory, sslParameters, hostnameVerifier);
  }
```

BinaryJedis里主要和redis Server进行交互，一系列Commands接口主要是对redis支持的接口进行分类，像BasicCommands主要包含了info、flush等操作，BinaryJedisCommands 主要包含了get、set等操作，MultiKeyBinaryCommands主要包含了一些批量操作的接口例如mset等。


JedisPool初始化

未完待续