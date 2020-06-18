---
title: Java并发编程之线程池
tags:
  - 并发
toc: true
categories:
  - 高并发
top: 'false'
cover: 'false'
img: '/images/2020/06/18/c2f89700-b146-11ea-a314-095877051747.png'
author: jjlu521016
date: 2020-06-18 18:19:00
---

## 什么是线程池： 

**java.util.concurrent.Executors提供了一个 java.util.concurrent.Executor接口的实现用于创建线程池**

## 为什么使用线程池

在说线程池之前，我们不得不提下线程。为什么有了线程还需要使用线程池技术呢？下面我们简单的了解下。

![image.png](/images/2020/06/18/c2f89700-b146-11ea-a314-095877051747.png)

**1. 创建/销毁线程伴随着系统开销，过于频繁的创建/销毁线程，会很大程度上影响处理效率**

例如：

创建线程消耗时间T1

执行任务消耗时间T2

销毁线程消耗时间T3

如果T1 + T3 远大于 T2，那么是不是说开启一个线程来执行这个任务代价有点大

2.**线程并发数量过多，抢占系统资源从而导致阻塞**

线程能共享系统资源，如果同时执行的线程过多，就有可能导致系统资源不足而产生阻塞的情况

3.**对线程进行一些简单的管理**

比如：延时执行、定时循环执行的策略

## 线程池的原理

<!-- more -->
在线程池中存在几个概念：

`核心线程数`指的是线程池的基本大小；

`最大线程数`指的是，同一时刻线程池中线程的数量最大不能超过该值；

`任务队列`是当任务较多时，线程池中线程的数量已经达到了核心线程数，这时候就是用任务队列来存储我们提交的任务。 

与其他池化技术不同的是，线程池是基于`生产者-消费者`模式来实现的，任务的提交方是生产者，线程池是消费者。当我们需要执行某个任务时，只需要把任务扔到线程池中即可。线程池中执行任务的流程如下图如下。

![image.png](/images/2020/06/18/d2a00170-b146-11ea-a314-095877051747.png)

## ThreadPoolExecutor简介

ThreadPoolExecutor

在JUC包下，已经提供了线程池的具体的实现：`ThreadPoolExecutor`。ThreadPoolExecutor提供了很多的构造方法，其中最复杂的构造方法有7个参数。

```
public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue,
                          ThreadFactory threadFactory,
                          RejectedExecutionHandler handler) {
}
```

- `corePoolSize`线程池的核心线程数。

  在任务提交到线程池的时候，

  线程数 <= corePoolSize 就会新创建的一个线程来执行任务

  线程数 > corePoolSize 就将任务添加到`workQueue`中。

- `maximumPoolSize`：线程池中允许存在的最大线程数量。

```
        当`workQueue`满了以后，再有新的任务进入到线程池时，会判断再新建一个线程是否会超过maximumPoolSize，
```

```
        如果会超过，则不创建线程，而是执行拒绝策略。
```

```
        如果不会超过maximumPoolSize，则会创建新的线程来执行任务。
```

- `keepAliveTime`：空闲线程等待工作的超时时间 

```
        当线程池中的线程数量大于corePoolSize时，大于corePoolSize这部分的线程如果没有任务去处理，那么就表示它们是空闲 
```

```
        的，这个时候是不允许它们一直存在的，而是允许它们最多空闲一段时间，这段时间就是keepAliveTime，时间的单位就是 
```

```
        TimeUnit。
```

- `unit`：空闲线程允许存活时间的单位
     `TimeUnit`是一个枚举值，它可以是纳秒、微妙、毫秒、秒、分、小时、天。
- `workQueue`：任务队列，用来存放任务。该队列的类型是阻塞队列。
      常用阻塞队列有`ArrayBlockingQueue、LinkedBlockingQueue、SynchronousQueue、PriorityBlockingQueue`等。 

```
  **关于阻塞队列下面会介绍隐藏的一些坑。**
```

- `threadFactory`：线程池工厂，用来创建线程。

```
          在实际项目中，为了便于后期排查问题，在创建线程时需要为线程赋予一定的名称
```

- `handler`：拒绝策略。
![image.png](/images/2020/06/18/e010dcd0-b146-11ea-a314-095877051747.png)
当任务队列已满，线程数量达到maximumPoolSize后，线程池就不会再接收新的任务了，这个时候就需要使用拒绝策略来决定最终是怎么处理这个任务。`默认情况下AbortPolicy，表示无法处理新任务，直接抛出异常`。在ThreadPoolExecutor类中定义了四个内部类，分别表示四种拒绝策略。我们也可以通过实现`RejectExecutionHandler`接口来实现自定义的拒绝策略。

`AbortPocily`：不再接收新任务，直接抛出异常。

`CallerRunsPolicy`：提交任务的线程自己处理。

`DiscardPolicy`：不处理，直接丢弃。

`DiscardOldestPolicy`：丢弃任务队列中排在最前面的任务，并执行当前任务。（排在队列最前面的任务并不一定是在队列中待的时间最长的任务，因为有可能是按照优先级排序的队列）

**`知识延伸`**,想深入理解的同学可以看下相关的源码可以加深印象。

`dubbo`中的线程池继承了`ThreadPoolExecutor.AbortPolicy`，重写了`rejectedExecution`方法，并且`dubbo`线程模型的所有拒绝策略都是使用的`AbortPolicyWithReport`。

`Netty`很像JDK中的CallerRunsPolicy，舍不得丢弃任务。不同的是，CallerRunsPolicy是直接在调用者线程执行的任务。而 Netty是新建了一个线程来处理的。

`ActiveMq`中的策略属于最大努力执行任务型，当触发拒绝策略时，在尝试一分钟的时间重新将任务塞进任务队列，当一分钟超时还没成功时，就抛出异常。

## Executors

Executors创建返回ThreadPoolExecutor对象的方法共有三种：

- Executors#newCachedThreadPool  创建可缓存的线程池
- Executors#newSingleThreadExecutor 创建单线程的线程池
- Executors#newFixedThreadPool 创建固定长度的线程池

看过《[**阿里巴巴开发规范**](https://yq.aliyun.com/attachment/download/?spm=a2c4e.11153940.0.0.49d73524UHXvow&id=5585)》的都知道，有这么一条

【强制】线程池不允许使用Executors去创建，而是通过ThreadPoolExecutor的方式。

下面我们根据相关源码来看下为什么有这条规约。

`CachedThreadPool`

```
   /**
     * corePoolSize => 0，核心线程池的数量为0
     * maximumPoolSize => Integer.MAX_VALUE，可以认为最大线程数是无限的
     * keepAliveTime => 60L
     * unit => 秒
     * workQueue => SynchronousQueue
     */
     public static ExecutorService newCachedThreadPool(ThreadFactory threadFactory) {
        return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                      60L, TimeUnit.SECONDS,
                                      new SynchronousQueue<Runnable>(),
                                      threadFactory);
    }
```

当一个任务提交时，corePoolSize为0不创建核心线程，SynchronousQueue是一个不存储元素的队列，可以理解为队里永远是满的，因此最终会创建非核心线程来执行任务。

对于非核心线程空闲60s时将被回收。因为Integer.MAX\_VALUE非常大 (2^32 -1)，**可以认为是可以无限创建线程的**，在资源有限的情况下容易引起OOM异常

`newSingleThreadExecutor` 

```
  /**
   * corePoolSize => 1，核心线程池的数量为1
   * maximumPoolSize => 1，只可以创建一个非核心线程
   * keepAliveTime => 0L
   * unit => 秒
   * workQueue => LinkedBlockingQueue
   */
  public static ExecutorService newSingleThreadExecutor(ThreadFactory threadFactory) {
        return new FinalizableDelegatedExecutorService
            (new ThreadPoolExecutor(1, 1,
                                    0L, TimeUnit.MILLISECONDS,
                                    new LinkedBlockingQueue<Runnable>(),
                                    threadFactory));
    }
```

当一个任务提交时，首先会创建一个核心线程来执行任务，如果超过核心线程的数量，将会放入队列中，因为**LinkedBlockingQueue是长度为Integer.MAX\_VALUE的队列**，可以认为是无界队列，因此往队列中可以插入无限多的任务，在资源有限的时候容易引起OOM异常，因为无界队列，**maximumPoolSize和keepAliveTime参数将无效。**

`newFixedThreadPool` 

```
   /**
     * corePoolSize => 1，核心线程池的数量为1
     * maximumPoolSize => 1，只可以创建一个非核心线程
     * keepAliveTime => 0L
     * unit => 秒
     * workQueue => LinkedBlockingQueue
     * LinkedBlockingQueue和SingleThreadExecutor类似，唯一的区别就是核心线程数不同，
     * 并且由于使用的是LinkedBlockingQueue，在资源有限的时候容易引起OOM异常
     */
 public static ExecutorService newFixedThreadPool(int nThreads, ThreadFactory threadFactory) {
        return new ThreadPoolExecutor(nThreads, nThreads,
                                      0L, TimeUnit.MILLISECONDS,
                                      new LinkedBlockingQueue<Runnable>(),
                                      threadFactory);
    }
```

综上所述：

`FixedThreadPool`和`SingleThreadExecutor` 

允许的请求队列长度为Integer.MAX\_VALUE，可能会堆积大量的请求，从而引起OOM异常

`CachedThreadPool` 

允许创建的线程数为Integer.MAX\_VALUE，可能会创建大量的线程，从而引起OOM异常

这就是为什么禁止使用Executors去创建线程池，而是推荐自己去创建ThreadPoolExecutor的原因