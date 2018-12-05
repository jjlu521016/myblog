---
title: 浅入Java ClassLoader
tags:
  - JVM
  - classloader
toc: true
categories:
  - java
date: 2018-12-05 10:04:08
---

# 1. ClassLoader是做什么的？
ClassLoader是用来加载Class 文件的。它负责将 Class 的字节码形式转换成内存形式的 Class 对象。字节码可以来自于磁盘文件 *.class，也可以是 jar 包里的 *.class，也可以来自远程服务器提供的字节流，字节码的本质就是一个字节数组 []byte，它有特定的复杂的内部格式。
<img src="/image/jvm/byteArr2Class.png">
# 2.ClassLoader的特点
<!-- more -->
## 2.1 延迟加载

JVM 运行的时候不是一次性把所有的类全部加载进来，它是按需加载（延迟加载）。程序在运行的时候会遇到一些新的类，在这个时候程序就会调用Classloader来加载这些类。加载完成将Class对象存放在Classloader中，下次再遇到这些类的时候就不需要重新加载进来了。

## 2.2 各司其职  
JVM 中内置三个重要的 ClassLoader:
- BootstrapClassLoader
- ExtensionClassLoader
- AppClassLoader。  

JVM 运行实例中会存在多个 ClassLoader，不同的 ClassLoader 会从不同的地方加载字节码文件。它可以从不同的文件目录加载，也可以从不同的 jar 文件中加载，也可以从网络上不同的静态文件服务器来下载字节码再加载。   

## 2.2.1 BootstrapClassLoader    
负责加载 JVM 运行时核心类，这些类位于 $JAVA_HOME/lib/rt.jar 文件中，我们常用内置库 java.xxx.* 都在里面，比如 java.util.、java.io.、java.nio.、java.lang. 等等。这个 ClassLoader 比较特殊，它是由 C 代码实现的，我们将它称之为「根加载器」。
## 2.2.2 ExtensionClassLoader   
负责加载 JVM 扩展类，比如 swing 系列、内置的 js 引擎、xml 解析器 等等，这些库名通常以 javax 开头，它们的 jar 包位于 $JAVA_HOME/lib/ext/*.jar 中，有很多 jar 包。
## 2.2.3 AppClassLoader  
才是直接面向用户的加载器，它会加载 Classpath 环境变量里定义的路径中的 jar 包和目录。我们自己编写的代码以及使用的第三方 jar 包通常都是由它来加载的。
- ClassLoader.getSystemClassLoader()
AppClassLoader 可以由 ClassLoader 类提供的静态方法 getSystemClassLoader() 得到，它就是我们所说的`系统类加载器`，我们用户平时编写的类代码通常都是由它加载的。当我们的 main 方法执行的时候，这第一个用户类的加载器就是 AppClassLoader。

## 2.2.4 URLClassLoader
jdk 内置了一个 URLClassLoader，对于网络上静态文件服务器提供的 jar 包和 .class文件，用户只需要传递规范的网络路径给构造器，就可以使用  URLClassLoader 来加载远程类库了。URLClassLoader 不但可以加载远程类库，还可以加载本地路径的类库，取决于构造器中不同的地址形式。ExtensionClassLoader 和 AppClassLoader 都是 URLClassLoader 的子类，它们都是从本地文件系统里加载类库。
<img src="/image/jvm/URLClassLoader.png">  

- 几种Classloader的配置参数  

| Class Loader类型 | 参数选项 | 说明  |
| ------ | ------  | ------  |		
|                     | -Xbootclasspath:              |	设置引导类加载器的搜索路径 |
|BootstrapClassLoader |	-Xbootclasspath/a:            |	把路径添加到已存在的搜索路径的后面 |
|                     |-Xbootclasspath/p:	            | 把路径添加到已存在的搜索路径的前面 |
|ExtClassLoader	      |-Djava.ext.dirs	              | 设置ExtClassLoader的搜索路径     |
|AppClassLoader	      |-Djava.class.path= 或-classpath| 设置AppClassLoader的搜索路径 |

# 2.3 传递性
程序在运行过程中，遇到了一个未知的类，它会选择哪个 ClassLoader 来加载它呢？    
虚拟机的策略是使用调用者 Class 对象的 ClassLoader 来加载当前未知的类。  
何为调用者 Class 对象？  
就是在遇到这个未知的类时，虚拟机肯定正在运行一个方法调用（静态方法或者实例方法），这个方法挂在哪个类上面，那这个类就是调用者 Class 对象。前面我们提到每个 Class 对象里面都有一个 classLoader 属性记录了当前的类是由谁来加载的。
因为 ClassLoader 的传递性，所有延迟加载的类都会由初始调用 main 方法的这个 ClassLoader 全全负责，它就是 AppClassLoader。

# 2.4 双亲委派
如果一个类加载器收到了类加载的请求，它首先不会自己尝试加载这个类，而是把这个请求委派给父类加载器去完成，每一个层次的类加载器都是如此，因此所有的加载请求最终请求都应该传送到顶层的启动类加载器中，只有当父加载器反馈自己无法完成这个加载请求(它的搜索范围中没有所需的类)时,子加载器才会自己尝试加载.Java类随着它的类加载器一起具备了一种带有优先级的层次关系。  
前面我们提到 AppClassLoader 只负责加载 Classpath 下面的类库，如果遇到没有加载的系统类库怎么办，AppClassLoader 必须将系统类库的加载工作交给 BootstrapClassLoader 和 ExtensionClassLoader 来做，这就是我们常说的「双亲委派」。
<img src="/image/jvm/shuangqin.png">
- 双亲委托模型的实现
```java
protected synchronized Class<?> loadClass ( String name , boolean resolve ) throws ClassNotFoundException{
        //检查指定类是否被当前类加载器加载过
        Class clazz = findLoadedClass(name);
        if( clazz == null ){//如果没被加载过，委派给父加载器加载
            try{
                if( parent != null )
                    clazz = parent.loadClass(name,resolve);
                else 
                   clazz = findBootstrapClassOrNull(name);
            }catch ( ClassNotFoundException e ){
                //如果父加载器无法加载
            }
            if( clazz == null ){//父类不能加载，由当前的类加载器加载
                clazz = findClass(name);
            }
        }
        if( resolve ){//如果要求立即链接，那么加载完类直接链接
            resolveClass();
        }
        //将加载过这个类对象直接返回
        return clazz;
    }
```
# 2.5 Class.forName 
forName 方法同样也是使用调用者 Class 对象的 ClassLoader 来加载目标类。不过 forName 还提供了多参数版本，可以指定使用哪个 ClassLoader 来加载
```java
Class<?> forName(String name, boolean initialize, ClassLoader cl)
```
通过这种形式的 forName 方法可以突破内置加载器的限制，通过使用自定类加载器允许我们自由加载其它任意来源的类库。根据 ClassLoader 的传递性，目标类库传递引用到的其它类库也将会使用自定义加载器加载。

# 2.6 自定义加载器
ClassLoader 里面有三个重要的方法 loadClass()、findClass() 和 defineClass()。
```java
//将byte字节流解析成JVM能够识别的Class对象，有了这个方法一位着我们
//不仅可以通过Class文件获得Class对象，其他的字节流都可以
Class<?> defineClass ( byte[] , int , int )
//实现类的加载规则，从而取得要加载类的字节码
Class<?> findClass(String)
//如果不想重新定义加载类额规则，也没有复杂的处理逻辑
//只是想能够一个加载一个自己指定的类，可以直接使用load
Class<?> loadClass(String)
//链接参数类，链接参照上面，不再赘述
void resolveClass(Class<?>)

```
- loadClass() 
该方法是加载目标类的入口，它首先会查找当前 ClassLoader 以及它的双亲里面是否已经加载了目标类，如果没有找到就会让双亲尝试加载，如果双亲都加载不了，就会调用 findClass() 让自定义加载器自己来加载目标类。ClassLoader 的 findClass() 方法是需要子类来覆盖的，不同的加载器将使用不同的逻辑来获取目标类的字节码。拿到这个字节码之后再调用 defineClass() 方法将字节码转换成 Class 对象。
```java 
class ClassLoader {

  // 加载入口，定义了双亲委派规则
  Class loadClass(String name) {
    // 是否已经加载了
    Class t = this.findFromLoaded(name);
    if(t == null) {
      // 交给双亲
      t = this.parent.loadClass(name)
    }
    if(t == null) {
      // 双亲都不行，只能靠自己了
      t = this.findClass(name);
    }
    return t;
  }
  
  // 交给子类自己去实现
  Class findClass(String name) {
    throw ClassNotFoundException();
  }
  
  // 组装Class对象
  Class defineClass(byte[] code, String name) {
    return buildClassFromCode(code, name);
  }
}

class CustomClassLoader extends ClassLoader {

  Class findClass(String name) {
    // 寻找字节码
    byte[] code = findCodeFromSomewhere(name);
    // 组装Class对象
    return this.defineClass(code, name);
  }
}

```
# 2.6 Class.forName vs ClassLoader.loadClass
这两个方法都可以用来加载目标类，它们之间有一个小小的区别，那就是 Class.forName() 方法可以获取原生类型的 Class，而 ClassLoader.loadClass() 则会报错。

