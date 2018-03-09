---
title: Dubbo Filter 传递上下文环境信息
toc: true
date: 2017-07-26 17:37:34
tags:
    - dubbo
---

### 需求
一般dubbo的service层都是一些通用的，无状态的服务。但是在某些特殊的需求下，需要传递一些上下文环境,打个不恰当的比方，例如需要在每次调用dubbo的服务的时候，记录一下用户名或者需要知道sessionid等。
<!-- more -->
### 解决办法1
如果是在项目设计的时候就意识到这一点的话，就好办，把所有的dubbo服务请求的参数都封装一个公共的父类，把一些上下文环境在放在父类的属性中。
![](http://upload-images.jianshu.io/upload_images/3353177-4178b9514ef69d51.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
这样做的好处就是，dubbo接口的参数都统一的，在Dubbo中可以做一些统一的处理（例如把上下文环境取出来，放在ThreadLocal中）。
#### 解决办法2
但是并不是所有的项目一开始就有这个需求的，但是突然有一天他猝不及防的出现了（比如本人就接到要使用多数据，每次前端请求的时候根据参数选择使用的数据库），如果项目已经基本定型的情况下，再改造成上面的解决办法，改动量太大（不怕麻烦的也可以，但是本人就比较懒）。
其实Dubbo的文档中已经有这个解决办法，就是隐式传参，
[http://dubbo.io/User+Guide-zh.htm#UserGuide-zh-%E9%9A%90%E5%BC%8F%E4%BC%A0%E5%8F%82](http://dubbo.io/User+Guide-zh.htm#UserGuide-zh-%E9%9A%90%E5%BC%8F%E4%BC%A0%E5%8F%82)
改造方案
代码如下
#### DubboConsumer:
```
@Activate(group = Constants.CONSUMER)
public class DubboConsumerContextFilter implements Filter {

    @Override
    public Result invoke(Invoker<?> invoker, Invocation invocation) throws RpcException {


        Map<String, String> attachments = new HashMap<>();
        attachments.put(xxx, xxx);
        attachments.put(xxx, xxx);
        //设置需要的内容
        RpcContext.getContext().setAttachments(attachments);
        return invoker.invoke(invocation);
    }
}
```

#### DubboProvider：

```
@Activate(group = Constants.PROVIDER)
public class DubboProviderContextFilter implements Filter {
    private Logger logger = LoggerFactory.getLogger(this.getClass());

    @Override
    public Result invoke(Invoker<?> invoker, Invocation invocation) throws RpcException {
        logger.info("DbProviderContextFilter attachments={}", RpcContext.getContext().getAttachments());

        String datasource = RpcContext.getContext().getAttachment(DbContextHolder.DATASOURCE_KEY);
        String schema = RpcContext.getContext().getAttachment(DbContextHolder.SCHEMA_KEY);
        if (StringUtils.isNotEmpty(datasource)) {
            DbContextHolder.setDatasource(datasource);
        }
        if (StringUtils.isNotEmpty(schema)) {
            DbContextHolder.setSchema(schema);
        }
        return invoker.invoke(invocation);
    }
}
```


第二步：在resources中创建文件
META-INF/dubbo/com.alibaba.dubbo.rpc.Filter

注意是 META-INF文件下的dubbo文件夹下的"com.alibaba.dubbo.rpc.Filter"文件
![](http://upload-images.jianshu.io/upload_images/3353177-01d142b5d2ef144f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
并在里面加入,也就是第一步中创建的类的路径
dubboContextFilter=com.xxx.DubboContextFilter
第三步：在配置文件中加入
<dubbo:provider filter="dubboContextFilter" />

#### 小结:
其实dubbo内置了一些filter，我们可以自定义自己的filter来完成一些和业务流程无关的逻辑，例如可以写IP白名单等等
