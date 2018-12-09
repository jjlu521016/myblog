---
title: Jedis源码解读-JedisPool
toc: true
date: 2018-12-09 22:26:46
tags:
---

# 1. 什么是对象池
对于一个对象，在其生命周期大致可以分为 `创建 -> 使用 -> 销毁`三大阶段，这个对象的时间是 T1(创建)+T2(使用)+T3(销毁)，
对于创建N个对象都需要这个步骤的话，肯定很耗时并且消耗性能的。
<!-- more -->
** 官方对于对象池的解释是：**
>　将用过的对象保存起来,等下次需要这种对象的时候再拿出来重复使用,从而在一定程度上减少频繁创建对象所造成的开销,用于充当保存对象的"容器"对象,被称为"对象池"。

# 2. JedisPool的创建
Jedis的连接池是基于apache.common.pool2,因此jedisPool的实现都是基于Pool2。
关于pool2的源码文档开业参考
http://commons.apache.org/proper/commons-pool/

<image src="/image/jedis/jedisPool.png"/>
JedisPool是common-pool2的Pool抽象类的子类。
```java
public class JedisPoolAbstract extends Pool<Jedis> {
...
}
```
其中JedisPool构造函数最终都是调用了Pool#initPool的方法
```java
package redis.clients.jedis.util;
public abstract class Pool<T> implements Closeable {
  protected GenericObjectPool<T> internalPool;

  /**
   * Using this constructor means you have to set and initialize the internalPool yourself.
   */
  public Pool() {
  }

  public Pool(final GenericObjectPoolConfig poolConfig, PooledObjectFactory<T> factory) {
    initPool(poolConfig, factory);
  }

  @Override
  public void close() {
    destroy();
  }

  public boolean isClosed() {
    return this.internalPool.isClosed();
  }

  public void initPool(final GenericObjectPoolConfig poolConfig, PooledObjectFactory<T> factory) {

    if (this.internalPool != null) {
      try {
        closeInternalPool();
      } catch (Exception e) {
      }
    }
    # 实例化pool2中的GenericeObjectPool
    this.internalPool = new GenericObjectPool<T>(factory, poolConfig);
  }
  # 从JedisPool获取Jedis和释放Jedis实例， 
  public T getResource() {
    try {
      # 阅读源码位置 org.apache.commons.pool2.impl#borrowObject
      return internalPool.borrowObject();
    } catch (NoSuchElementException nse) {
      if (null == nse.getCause()) { // The exception was caused by an exhausted pool
        throw new JedisExhaustedPoolException(
            "Could not get a resource since the pool is exhausted", nse);
      }
      // Otherwise, the exception was caused by the implemented activateObject() or ValidateObject()
      throw new JedisException("Could not get a resource from the pool", nse);
    } catch (Exception e) {
      throw new JedisConnectionException("Could not get a resource from the pool", e);
    }
  }

  protected void returnResourceObject(final T resource) {
    if (resource == null) {
      return;
    }
    try {
      internalPool.returnObject(resource);
    } catch (Exception e) {
      throw new JedisException("Could not return the resource to the pool", e);
    }
  }
```
initPool方法中有两个参数：一个GenericObjectPoolConfig配置项分装类，一个是 PooledObjectFactory工厂类获取到连接池后从连接池中获取jedis连接对象。
根据配置信息实例化pool2中的GenericeObjectPool这个对象池的管理者。

当调用 getResource 获取Jedis时， 实际上是Pool内部的internalPool调用borrowObject()拿到一个实例 ，而internalPool 这个 GenericObjectPool 又调用了 JedisFactory 的 makeObject() 来完成实例的生成 (在Pool中资源不够的时候)


# Jedis是如何实例化的
我们先看下JedisFactory的源码
```java
class JedisFactory implements PooledObjectFactory<Jedis> {
  @Override
  public PooledObject<Jedis> makeObject() throws Exception {
    final HostAndPort hostAndPort = this.hostAndPort.get();
    final Jedis jedis = new Jedis(hostAndPort.getHost(), hostAndPort.getPort(), connectionTimeout,
        soTimeout, ssl, sslSocketFactory, sslParameters, hostnameVerifier);

    try {
      jedis.connect();
      if (password != null) {
        jedis.auth(password);
      }
      if (database != 0) {
        jedis.select(database);
      }
      if (clientName != null) {
        jedis.clientSetname(clientName);
      }
    } catch (JedisException je) {
      jedis.close();
      throw je;
    }

    return new DefaultPooledObject<Jedis>(jedis);

  }
...
}
```
JedisFactory实现了pool2的PooledObjectFactory接口,池中对象创建和销毁的接口，交由业务方（就是文中的JedisFactory来实现））。
在JedisFactory#makeObject()方法中创建了Jedis对象。
Jedis继承自BinaryJedis，其有一个Client属性，Client是Connection的子类，Connection中有socket这个属性，也就是真正跟redis服务端创建连接的类，并且这个socket是个长连接。
<image src="/image/jedis/jedis.png"/>
`redis.clients.jedis.Connection#connect
```java
public void connect() {
    if (!isConnected()) {
      try {
        socket = new Socket();
        // ->@wjw_add
        socket.setReuseAddress(true);
        #建立长连接
        socket.setKeepAlive(true); // Will monitor the TCP connection is
        // valid
        socket.setTcpNoDelay(true); // Socket buffer Whetherclosed, to
        // ensure timely delivery of data
        socket.setSoLinger(true, 0); // Control calls close () method,
        // the underlying socket is closed
        // immediately
        // <-@wjw_add

        socket.connect(new InetSocketAddress(host, port), connectionTimeout);
        socket.setSoTimeout(soTimeout);
        ......
        outputStream = new RedisOutputStream(socket.getOutputStream());
        inputStream = new RedisInputStream(socket.getInputStream());
      } catch (IOException ex) {
        broken = true;
        throw new JedisConnectionException("Failed connecting to host " 
            + host + ":" + port, ex);
      }
    }
  }
  ```
  从上面的源码可以看到，Jedis和网络连接是一一绑定的，假如redis对象被GC了，那么它的client和socket连接一并销毁。

  # 客户端归还对象池
  归还池的处理规则是由common-pool2来实现的，就是把jedis对象放到空闲队列中，如果队列满了，就将其直接销毁，销毁是在JedisFactory实现的
  redis.clients.jedis.JedisFactory#destroyObject
```java

  @Override
  public void destroyObject(PooledObject<Jedis> pooledJedis) throws Exception {
    final BinaryJedis jedis = pooledJedis.getObject();
    if (jedis.isConnected()) {
      try {
        try {
          jedis.quit();
        } catch (Exception e) {
        }
        jedis.disconnect();
      } catch (Exception e) {

      }
    }

  }
  ```
  当主动关闭socket连接，在common-pool2中的GenericObjectPool也会把他从空闲池和总池中移除。

