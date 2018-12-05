---
title: ThreadLocal的使用及原理分析
tags: [ 
    -java
    -ThreadLocal
]
toc: true
categories: [java]
date: 2018-12-05 21:50:31
---

# 1. 什么是ThreadLocal？
ThreadLocal称作线程本地存储。简单来说，就是ThreadLocal为共享变量在每个线程中都创建一个副本，每个线程可以访问自己内部的副本变量。这样做的好处是可以保证共享变量在`多线程环境下访问的线程安全性`。

# 2. ThreadLocal的使用
引题：
在没有使用ThreadLocal的时候，定义了一个静态的成员变量num，然后通过构造5个线程对这个num做递增
<!-- more -->
```java
    private static Integer num=0;

    public static void main(String[] args) {
        Thread[] threads=new Thread[5];
        for(int i=0;i<5;i++){
            threads[i]=new Thread(()->{
               num+=5;
               System.out.println(Thread.currentThread().getName()+" : "+num);
            },"Thread-"+i);
        }

        for(Thread thread:threads){
            thread.start();
        }
    }
```
执行这个方法之后，你会发现每次的运行结果都是不一样的。

使用了ThreadLocal以后：
```java
  private static final ThreadLocal<Integer> local=new ThreadLocal<Integer>(){
        protected Integer initialValue(){
	   //通过initialValue方法设置默认值
            return 0; 
        }
    };

    public static void main(String[] args) {
        Thread[] threads=new Thread[5];
        for(int i=0;i<5;i++){
            threads[i]=new Thread(()->{
                int num=local.get().intValue();
                num+=5;
               System.out.println(Thread.currentThread().getName()+" : "+num);
            },"Thread-"+i);
        }

        for(Thread thread:threads){
            thread.start();
        }
    }
```
从结果可以看到，每个线程的值都是5，意味着各个线程之间都是独立的变量副本，彼此不相互影响.

> ThreadLocal会给定一个初始值，也就是initialValue()方法，而每个线程都会从ThreadLocal中获得这个初始化的值的副本，这样可以使得每个线程都拥有一个副本拷贝。  

从ThreadLocal的方法定义来看，就几个方法
- get: 获取ThreadLocal中当前线程对应的线程局部变量
- set：设置当前线程的线程局部变量的值
- remove：将当前线程局部变量的值删除  
- initialValue 返回当前线程局部变量的初始值  
<img src="/image/java/threadlocal.png">
> 源码基于JDK 1.8   

get方法的实现
```java
 public T get() {
        // 当前执行的线程
        Thread t = Thread.currentThread();
        // 获得当前线程的ThreadLocalMap实例
        ThreadLocalMap map = getMap(t);
        // 如果map不为空，说明当前线程已经有了一个ThreadLocalMap实例
        if (map != null) {
	    # 获取Entry
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null) {
                @SuppressWarnings("unchecked")
                T result = (T)e.value;
                return result;
            }
        }
        return setInitialValue();
    }

  /**
     * Variant of set() to establish initialValue. Used instead
     * of set() in case user has overridden the set() method.
     *
     * @return the initial value
     */
    private T setInitialValue() {
        T value = initialValue();
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
             //如果map不为null,把初始化value设置进去
            map.set(this, value);
        else
            //如果map为null,则new一个map,并把初始化value设置进去
            createMap(t, value);
        return value;
    }
```

set方法的实现
```java
public void set(T value) {
       
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t); 
        if (map != null)
            // 直接将当前value设置到ThreadLocalMap中
            map.set(this, value);
        else
            // 说明当前线程是第一次使用线程本地变量，构造map
            createMap(t, value); 
    }
```
从上面方法中，我们可以注意到有一个叫做ThreadLocalMap 的对象，它是做什么的呢？

ThreadLocalMap是一个静态内部类，内部定义了一个Entry对象用来真正存储数据
```java
static class ThreadLocalMap {
        static class Entry extends WeakReference<ThreadLocal<?>> {
            /** The value associated with this ThreadLocal. */
            Object value;

            Entry(ThreadLocal<?> k, Object v) {
                super(k);
                value = v;
            }
        }
        ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
            //构造一个Entry数组，并设置初始大小
            table = new Entry[INITIAL_CAPACITY];
            //计算Entry数据下标
            int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
            //将`firstValue`存入到指定的table下标中
            table[i] = new Entry(firstKey, firstValue);
            size = 1;//设置节点长度为1
            setThreshold(INITIAL_CAPACITY); //设置扩容的阈值
        }
    //...省略部分代码
}
```
通过以上的代码我们又发现了新的大陆：

- Entry继承了WeakReference,这个表示什么意思?
- 在构造ThreadLocalMap的时候new ThreadLocalMap(this, firstValue);
key其实是this，this表示当前对象的引用，在当前的案例中，this指的是ThreadLocal<Integer> local。那么多个线程对应同一个ThreadLocal实例，怎么对每一个ThreadLocal对象做区分呢？

weakReference表示弱引用，在Java中有四种引用类型，强引用、弱引用、软引用、虚引用。
使用弱引用的对象，不会阻止它所指向的对象被垃圾回收器回收。
对于下面的示例代码我们分析下：
```java
DemoA a=new DemoA();
DemoB b=new DemoB(a);
a=null;
```
看似很正常的一段代码，但是将a对象的引用设置为null，当一个对象不再被其他对象引用的时候，是会被GC回收的，但是对于这个场景来说，即时是a=null，也不可能被回收，因为DemoB依赖DemoA，这个时候是可能造成内存泄漏的

接着看下面的代码
```java
//方法1
DemoA a=new DemoA();
DemoB b=new DemoB(a);
a=null;
b=null;
//方法2
DemoA a=new DemoA();
WeakReference b=new WeakReference(a);
a=null;
```
对于方法2来说，DemoA只是被弱引用依赖，假设垃圾收集器在某个时间点决定一个对象是弱可达的(weakly reachable)（也就是说当前指向它的全都是弱引用），这时垃圾收集器会清除所有指向该对象的弱引用，然后把这个弱可达对象标记为可终结(finalizable)的，这样它随后就会被回收。

> 试想一下如果这里没有使用弱引用，意味着ThreadLocal的生命周期和线程是强绑定，只要线程没有销毁，那么ThreadLocal一直无法回收。而使用弱引用以后，当ThreadLocal被回收时，由于Entry的key是弱引用，不会影响ThreadLocal的回收防止内存泄漏，同时，在后续的源码分析中会看到，ThreadLocalMap本身的垃圾清理会用到这一个好处，方便对无效的Entry进行回收

ThreadLocalMap以this作为key
在构造ThreadLocalMap时，使用this作为key来存储，那么对于同一个ThreadLocal对象，如果同一个Thread中存储了多个值，是如何来区分存储的呢？
答案就在firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1)
我们看下相关的源码
```java
void createMap(Thread t, T firstValue) {
        t.threadLocals = new ThreadLocalMap(this, firstValue);
}
/**
 * Construct a new map initially containing (firstKey, firstValue).
 * ThreadLocalMaps are constructed lazily, so we only create
 * one when we have at least one entry to put in it.
 */
 ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
     table = new Entry[INITIAL_CAPACITY];
     int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
     table[i] = new Entry(firstKey, firstValue);
     size = 1;
     setThreshold(INITIAL_CAPACITY);
}
```

关键点就在threadLocalHashCode，它相当于一个ThreadLocal的ID，实现的逻辑如下
```java
private final int threadLocalHashCode = nextHashCode();
private static AtomicInteger nextHashCode =
        new AtomicInteger();
private static final int HASH_INCREMENT = 0x61c88647;

private static int nextHashCode() {
    return nextHashCode.getAndAdd(HASH_INCREMENT);
}
```

这里用到了一个非常完美的散列算法，可以简单理解为，对于同一个ThreadLocal下的多个线程来说，当任意线程调用set方法存入一个数据到Entry中的时候，其实会根据threadLocalHashCode生成一个唯一的id标识对应这个数据，存储在Entry数据下标中。

- threadLocalHashCode是通过nextHashCode.getAndAdd(HASH_INCREMENT)来实现的  

i*HASH_INCREMENT+HASH_INCREMENT,每次新增一个元素(ThreadLocal)到Entry[],都会自增0x61c88647,目的为了让哈希码能均匀的分布在2的N次方的数组里

- Entry[i]= hashCode & (length-1)  

# 0x61c88647
```java
private static final int HASH_INCREMENT = 0x61c88647;
    public static void main(String[] args) {
        magicHash(16); //初始大小16
        magicHash(32); //扩容一倍
    }

    private static void magicHash(int size){
        int hashCode = 0;
        for(int i=0;i<size;i++){
            hashCode = i*HASH_INCREMENT+HASH_INCREMENT;
            System.out.print((hashCode & (size-1))+" ");
        }
        System.out.println();
    }
```
观察运行的结果找规律

魔数0x61c88647的选取和斐波那契散列有关，0x61c88647对应的十进制为1640531527。而斐波那契散列的乘数可以用(long) ((1L << 31) * (Math.sqrt(5) - 1)); 如果把这个值给转为带符号的int，则会得到-1640531527。也就是说
(long) ((1L << 31) * (Math.sqrt(5) - 1));得到的结果就是1640531527，也就是魔数0x61c88647

> 总结，我们用0x61c88647作为魔数累加为每个ThreadLocal分配各自的ID也就是threadLocalHashCode再与2的幂取模，得到的结果分布很均匀。 

## ThreadLocalMap.set(key,value)
```java
private void set(ThreadLocal<?> key, Object value) {
            Entry[] tab = table;
            int len = tab.length;
            // 根据哈希码和数组长度求元素放置的位置，即数组下标
            int i = key.threadLocalHashCode & (len-1);
             //从i开始往后一直遍历到数组最后一个Entry(线性探索)
            for (Entry e = tab[i];
                 e != null;
                 e = tab[i = nextIndex(i, len)]) {
                ThreadLocal<?> k = e.get();
                 //如果key相等，覆盖value
                if (k == key) {
                    e.value = value;
                    return;
                }
                 //如果key为null,用新key、value覆盖，同时清理历史key=null的陈旧数据
                if (k == null) {
                    replaceStaleEntry(key, value, i);
                    return;
                }
            }

            tab[i] = new Entry(key, value);
            int sz = ++size;
             //如果超过阀值，就需要扩容了
            if (!cleanSomeSlots(i, sz) && sz >= threshold)
                rehash();
        }
```
为了更直观的体现set方法的实现，通过一个图形表示如下
<img src="/image/java/threadlocal2.png">


# ThreadLocal的内存泄漏
ThreadLocalMap中Entry的key使用的是ThreadLocal的弱引用，如果一个ThreadLocal没有外部强引用，当系统执行GC时，这个ThreadLocal势必会被回收，这样一来，ThreadLocalMap中就会出现一个key为null的Entry，而这个key=null的Entry是无法访问的，当这个线程一直没有结束的话，那么就会存在一条强引用链

Thread Ref - > Thread -> ThreadLocalMap - > Entry -> value 永远无法回收而造成内存泄漏
<img src="/image/java/threadlocal3.png">
> 其实我们从源码分析可以看到，ThreadLocalMap是做了防护措施的
- 首先从ThreadLocal的直接索引位置(通过ThreadLocal.threadLocalHashCode & (len-1)运算得到)获取Entry e，如果e不为null并且key相同则返回e
- 如果e为null或者key不一致则向下一个位置查询，如果下一个位置的key和当前需要查询的key相等，则返回对应的Entry，否则，如果key值为null，则擦除该位置的Entry，否则继续向下一个位置查询  

在这个过程中遇到的key为null的Entry都会被擦除，那么Entry内的value也就没有强引用链，自然会被回收。仔细研究代码可以发现，set操作也有类似的思想，将key为null的这些Entry都删除，防止内存泄露。
但是这个设计一来与一个前提条件，就是调用get或者set方法，但是不是所有场景都会满足这个场景的，所以为了避免这类的问题，我们可以在合适的位置手动调用ThreadLocal的remove函数删除不需要的ThreadLocal，防止出现内存泄漏

所以建议的使用方法是
- 将ThreadLocal变量定义成private static的，这样的话ThreadLocal的生命周期就更长，由于一直存在ThreadLocal的强引用，所以ThreadLocal也就不会被回收，也就能保证任何时候都能根据ThreadLocal的弱引用访问到Entry的value值，然后remove它，防止内存泄露
- 每次使用完ThreadLocal，都调用它的remove()方法，清除数据