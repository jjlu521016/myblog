---
title: >-
  使用Jedis在高并发报错 (java.net.SocketException: Connection reset by peer: socket
  write error)
date: 2018.01.17 14:29:15
tags: jedis
toc: true
---

Connection reset by peer: socket write error错误分析：
常出现的Connection reset by peer: 原因可能是多方面的，不过更常见的原因是：  
①：服务器的并发连接数超过了其承载量，服务器会将其中一些连接Down掉；  
②：客户关掉了浏览器，而服务器还在给客户端发送数据；  
③：浏览器端按了Stop 
<!-- more -->

#### 1.报错信息
```log
java.lang.reflect.InvocationTargetException: null
	at sun.reflect.GeneratedMethodAccessor15.invoke(Unknown Source)
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(Unknown Source)
	at java.lang.reflect.Method.invoke(Unknown Source)
	....
	at java.lang.Thread.run(Unknown Source)
Caused by: redis.clients.jedis.exceptions.JedisConnectionException: java.net.SocketException: Connection reset by peer: socket write error
	at redis.clients.jedis.Connection.flush(Connection.java:334)
	at redis.clients.jedis.Connection.getBinaryBulkReply(Connection.java:257)
	at redis.clients.jedis.BinaryJedis.get(BinaryJedis.java:244)
	......
	... 15 common frames omitted
Caused by: java.net.SocketException: Connection reset by peer: socket write error
	at java.net.SocketOutputStream.socketWrite0(Native Method)
	at java.net.SocketOutputStream.socketWrite(Unknown Source)
	at java.net.SocketOutputStream.write(Unknown Source)
	at redis.clients.util.RedisOutputStream.flushBuffer(RedisOutputStream.java:52)
	at redis.clients.util.RedisOutputStream.flush(RedisOutputStream.java:216)
	at redis.clients.jedis.Connection.flush(Connection.java:331)
	... 22 common frames omitted
 ```
 所以本问题是由 ①造成的


 #### 2.修改之前的代码  
 初始化jedis的代码 
 ```java

   /**
     * 在多线程环境同步初始化
     */
    private static synchronized void poolInit() {
        if (pool == null) {
            createJedisPool();
        }

    }

   * 获取一个jedis 对象
     *
     * @return
     */
    public static Jedis getJedis() {

        if (pool == null) {
            poolInit();
        }
    
        return pool.getResource();
       
         
    }

 ```

 使用jedis的代码
 ```java
  private static Jedis jedis = JedisPoolUtil.getJedis();
  
  public static Object getObject(String key) {
        if(exists(key)){
            return deserialize(jedis.get(key.getBytes()));
        }
        return null;
    }
 ```

 #### 3.修改之后的代码
 初始化jedis的代码
 ```java
    /**
     * 获取一个jedis 对象
     *
     * @return
     */
    public static Jedis getJedis() {

        if (pool == null) {
            poolInit();
        }
        //如果没有以下代码会造成初始化的jedis拿不到 jedis对象
        Jedis jedis = null;
        try {
            if (pool != null) {
                jedis = pool.getResource();
            }
        }
        catch (Exception e) {
            logger.error("获取redis失败 : {}" + ExceptionUtils.getStackTrace(e));
        }
        return jedis;
    }
 ```
 使用jedis的代码
 ```java
 /**
     * 读取对象
     *
     * @param key
     * @return
     */
    public static Object getObject(String key) {
        if (exists(key)) {
            //初始化jedis用完之后关闭连接
            Jedis jedis = JedisPoolUtil.getJedis();
            Object object = deserialize(jedis.get(key.getBytes()));
            jedis.close();
            return object;
        }
        return null;
    }
```    