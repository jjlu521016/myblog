---
title: jmap命令使用
toc: true
date: 2018-04-19 16:22:48
tags:
    - jvm
    - jmap
---

### 查看 JVM 堆内存情况
如果想分析自己的JAVA Application时，可以使用jmap程序来生成heapdump文例：
`jmap -heap pid`
jmap是JDK自带的一个工具，非常小巧方便，其支持参数如下：
    `-heap`：打印heap空间的概要，这里可以粗略的检验heap空间的使用情况。
官网对jmap的解释是:
<!-- more -->
DESCRIPTION
```
jmap prints shared object memory maps or heap memory details of a given process or core file or a remote debug server. If the given process is running on a 64-bit VM, you may need to specify the -J-d64 option, e.g.:

jmap -J-d64 -heap pid
NOTE - This utility is unsupported and may or may not be available in future versions of the JDK. In Windows Systems where dbgeng.dll is not present, 'Debugging Tools for Windows' needs to be installed to have these tools working. Also, the PATH environment variable should contain the location of jvm.dll used by the target process or the location from which the Crash Dump file was produced.
For example, set PATH=<jdk>\jre\bin\client;%PATH%
```

OPTIONS
```
<no option>
When no option is used jmap prints shared object mappings. For each shared object loaded in the target VM, start address, the size of the mapping, and the full path of the shared object file are printed. This is similar to the Solaris pmap utility.

-dump:[live,]format=b,file=<filename>
Dumps the Java heap in hprof binary format to filename. The live suboption is optional. If specified, only the live objects in the heap are dumped. To browse the heap dump, you can use jhat (Java Heap Analysis Tool) to read the generated file.

-finalizerinfo
Prints information on objects awaiting finalization.

-heap
Prints a heap summary. GC algorithm used, heap configuration and generation wise heap usage are printed.

-histo[:live]
Prints a histogram of the heap. For each Java class, number of objects, memory size in bytes, and fully qualified class names are printed. VM internal class names are printed with '*' prefix. If the live suboption is specified, only live objects are counted.

-permstat
Prints class loader wise statistics of permanent generation of Java heap. For each class loader, its name, liveness, address, parent class loader, and the number and size of classes it has loaded are printed. In addition, the number and size of interned Strings are printed.

-F
Force. Use with jmap -dump or jmap -histo option if the pid does not respond. The live suboption is not supported in this mode.

-h
Prints a help message.

-help
Prints a help message.

-J<flag>
Passes <flag> to the Java virtual machine on which jmap is run.

```

```
对于jmap -heap命令显示的结果对应的含义如下:
```sh
jmap -heap 837
```
```properties
Attaching to process ID 837, please wait...
Debugger attached successfully.
Server compiler detected.
JVM version is 24.71-b01
using thread-local object allocation.
Parallel GC with 4 thread(s)//GC 方式
Heap Configuration: //堆内存初始化配置
   MinHeapFreeRatio = 0 //对应jvm启动参数-XX:MinHeapFreeRatio设置JVM堆最小空闲比率(default 40)
   MaxHeapFreeRatio = 100 //对应jvm启动参数 -XX:MaxHeapFreeRatio设置JVM堆最大空闲比率(default 70)
   MaxHeapSize      = 2082471936 (1986.0MB) //对应jvm启动参数-XX:MaxHeapSize=设置JVM堆的最大大小
   NewSize          = 1310720 (1.25MB)//对应jvm启动参数-XX:NewSize=设置JVM堆的‘新生代’的默认大小
   MaxNewSize       = 17592186044415 MB//对应jvm启动参数-XX:MaxNewSize=设置JVM堆的‘新生代’的最大大小
   OldSize          = 5439488 (5.1875MB)//对应jvm启动参数-XX:OldSize=<value>:设置JVM堆的‘老生代’的大小
   NewRatio         = 2 //对应jvm启动参数-XX:NewRatio=:‘新生代’和‘老生代’的大小比率
   SurvivorRatio    = 8 //对应jvm启动参数-XX:SurvivorRatio=设置年轻代中Eden区与Survivor区的大小比值 
   PermSize         = 21757952 (20.75MB)  //对应jvm启动参数-XX:PermSize=<value>:设置JVM堆的‘永生代’的初始大小
   MaxPermSize      = 85983232 (82.0MB)//对应jvm启动参数-XX:MaxPermSize=<value>:设置JVM堆的‘永生代’的最大大小
   G1HeapRegionSize = 0 (0.0MB)
Heap Usage://堆内存使用情况
PS Young Generation
Eden Space://Eden区内存分布
   capacity = 33030144 (31.5MB)//Eden区总容量
   used     = 1524040 (1.4534378051757812MB)  //Eden区已使用
   free     = 31506104 (30.04656219482422MB)  //Eden区剩余容量
   4.614088270399305% used //Eden区使用比率
From Space:  //其中一个Survivor区的内存分布
   capacity = 5242880 (5.0MB)
   used     = 0 (0.0MB)
   free     = 5242880 (5.0MB)
   0.0% used
To Space:  //另一个Survivor区的内存分布
   capacity = 5242880 (5.0MB)
   used     = 0 (0.0MB)
   free     = 5242880 (5.0MB)
   0.0% used
PS Old Generation //当前的Old区内存分布
   capacity = 86507520 (82.5MB)
   used     = 0 (0.0MB)
   free     = 86507520 (82.5MB)
   0.0% used
PS Perm Generation//当前的 “永生代” 内存分布
   capacity = 22020096 (21.0MB)
   used     = 2496528 (2.3808746337890625MB)
   free     = 19523568 (18.619125366210938MB)
   11.337498256138392% used
670 interned Strings occupying 43720 bytes.
```