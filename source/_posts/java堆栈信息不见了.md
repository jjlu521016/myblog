---
title: java堆栈信息不见了
toc: true
date: 2018-12-12 12:43:31
tags: 
    - jvm
---
# 1. 问题描述
最近同事通过ELK查找异常日志发现,exception的栈不见了，如下所示：
```
异常信息:java.lang.NullPointerException
异常信息:java.lang.NullPointerException
异常信息:java.lang.NullPointerException
```
<!-- more -->
本地试了很多次一直都能打印出异常信息，那么前面那段只有简单的java.lang.NullPointerException，没有详细异常栈信息的原因是什么呢？于是他问怎么出现这个现象的，我跟他说这种情况是 JVM对一些特定的异常类型做了Fast Throw优化导致的
```
java.lang.NullPointerException
    ...
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1149)
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624)
	at java.lang.Thread.run(Thread.java:748)
```

# 2. 什么是Fast Throw
JVM中有个参数：OmitStackTraceInFastThrow，就是省略异常栈信息将异常快速抛出。
## 2.1 JVM是如何做到快速抛出的呢？
JVM对一些特定的异常类型做了Fast Throw优化，如果检测到在代码里某个位置连续多次抛出同一类型异常的话，C2会决定用Fast Throw方式来抛出异常，而异常Trace即详细的异常栈信息会被清空。这种异常抛出速度非常快，因为不需要在堆里分配内存，也不需要构造完整的异常栈信息。相关的源码的JVM源码的graphKit.cpp文件中
源码地址
http://hg.openjdk.java.net/jdk8u/jdk8u/hotspot/file/tip/src/share/vm/opto/graphKit.cpp(从514行开始)
## 2.2 Fast Throw源码解析
```cpp

//------------------------------builtin_throw----------------------------------
void GraphKit::builtin_throw(Deoptimization::DeoptReason reason, Node* arg) {
  bool must_throw = true;

  ... ...
  // If this particular condition has not yet happened at this
  // bytecode, then use the uncommon trap mechanism, and allow for
  // a future recompilation if several traps occur here.
  // If the throw is hot, try to use a more complicated inline mechanism
  // which keeps execution inside the compiled code.
  bool treat_throw_as_hot = false;

  if (ProfileTraps) {
    if (too_many_traps(reason)) {
      treat_throw_as_hot = true;
    }
    // (If there is no MDO at all, assume it is early in
    // execution, and that any deopts are part of the
    // startup transient, and don't need to be remembered.)

    // Also, if there is a local exception handler, treat all throws
    // as hot if there has been at least one in this method.
    if (C->trap_count(reason) != 0
        && method()->method_data()->trap_count(reason) != 0
        && has_ex_handler()) {
        treat_throw_as_hot = true;
    }
  }

  // If this throw happens frequently, an uncommon trap might cause
  // a performance pothole.  If there is a local exception handler,
  // and if this particular bytecode appears to be deoptimizing often,
  // let us handle the throw inline, with a preconstructed instance.
  // Note:   If the deopt count has blown up, the uncommon trap
  // runtime is going to flush this nmethod, not matter what.
  // 满足两个条件：1.检测到频繁抛出异常，2. OmitStackTraceInFastThrow为true，或StackTraceInThrowable为false
  if (treat_throw_as_hot
      && (!StackTraceInThrowable || OmitStackTraceInFastThrow)) {
    // If the throw is local, we use a pre-existing instance and
    // punt on the backtrace.  This would lead to a missing backtrace
    // (a repeat of 4292742) if the backtrace object is ever asked
    // for its backtrace.
    // Fixing this remaining case of 4292742 requires some flavor of
    // escape analysis.  Leave that for the future.
    ciInstance* ex_obj = NULL;
    switch (reason) {
    case Deoptimization::Reason_null_check:
      ex_obj = env()->NullPointerException_instance();
      break;
    case Deoptimization::Reason_div0_check:
      ex_obj = env()->ArithmeticException_instance();
      break;
    case Deoptimization::Reason_range_check:
      ex_obj = env()->ArrayIndexOutOfBoundsException_instance();
      break;
    case Deoptimization::Reason_class_check:
      if (java_bc() == Bytecodes::_aastore) {
        ex_obj = env()->ArrayStoreException_instance();
      } else {
        ex_obj = env()->ClassCastException_instance();
      }
      break;
    }
    ... ...
}
```
从源码可以看到默认对下面的Exception都做了Fast Throw。
- NullPointerException
- ArithmeticException
- ArrayIndexOutOfBoundsException
- ArrayStoreException
- ClassCastException
> OmitStackTraceInFastThrow和StackTraceInThrowable都默认为true，所以条件(!StackTraceInThrowable || OmitStackTraceInFastThrow)为true，即JVM默认开启了Fast Throw优化。

如果想关闭Fast Throw的优化，在启动参数加上配置
-XX:-OmitStackTraceInFastThrow，
StackTraceInThrowable保持默认配置即可。

# 3.验证结果
写一个简单的代码验证
```java
public class JavaNPE extends Thread {
    private static int count = 0;
    @Override
    public void run() {
        try {
            System.out.println("getSimpleName is:"+this.getClass().getSimpleName() + " execute count：" + (++count));
            String str = null;
            System.out.println(str.length());
        } catch (Throwable e) {
            e.printStackTrace();
        }
    }
}

public class TestFastThrow {
    public static void main(String[] args) throws InterruptedException {
        JavaNPE javaNPE = new JavaNPE();
        ExecutorService executorService = Executors.newFixedThreadPool(10);
        for (int i = 0; i < Integer.MAX_VALUE; i++) {
            executorService.execute(javaNPE);
            //防止打出的日志太快
            Thread.sleep(2);
        }
    }
}
```
在没有加入`-XX:-OmitStackTraceInFastThrow`测试结果(NPE连续抛出5000+次就会丢失异常栈):
```
getSimpleName is:JavaNPE execute count：6666
java.lang.NullPointerException
	at com.jason.demo.demo.JavaNPE.run(JavaNPE.java:18)
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1149)
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624)
	at java.lang.Thread.run(Thread.java:748)
getSimpleName is:JavaNPE execute count：6667
java.lang.NullPointerException
	at com.jason.demo.demo.JavaNPE.run(JavaNPE.java:18)
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1149)
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624)
	at java.lang.Thread.run(Thread.java:748)
java.lang.NullPointerExceptiongetSimpleName is:JavaNPE execute count：6668

	at com.jason.demo.demo.JavaNPE.run(JavaNPE.java:18)
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1149)
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624)
	at java.lang.Thread.run(Thread.java:748)
getSimpleName is:JavaNPE execute count：6669
java.lang.NullPointerException
getSimpleName is:JavaNPE execute count：6670
java.lang.NullPointerException
getSimpleName is:JavaNPE execute count：6671
java.lang.NullPointerException
getSimpleName is:JavaNPE execute count：6672
java.lang.NullPointerException

```

加入`-XX:-OmitStackTraceInFastThrow`测试结果:
```
getSimpleName is:JavaNPE execute count：11325
java.lang.NullPointerException
	at com.jason.demo.demo.JavaNPE.run(JavaNPE.java:18)
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1149)
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624)
	at java.lang.Thread.run(Thread.java:748)
getSimpleName is:JavaNPE execute count：11326
java.lang.NullPointerException
	at com.jason.demo.demo.JavaNPE.run(JavaNPE.java:18)
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1149)
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624)
	at java.lang.Thread.run(Thread.java:748)
getSimpleName is:JavaNPE execute count：11327
```

参考：
https://www.oracle.com/technetwork/java/javase/relnotes-139183.html
https://stackoverflow.com/questions/2411487/nullpointerexception-in-java-with-no-stacktrace