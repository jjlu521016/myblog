---
title: elk6.x 安装x-pack
toc: true
date: 2018-08-22 13:39:50
tags:
    -elk
---
注意：安装x-pack的时候一定要停止对应的服务，不然可能出现各种异常报错
## elasticsearch安装x-pack
切换到es的bin目录
```sh
./elasticsearch-plugin install x-pack
```
稍等片刻可以安装完毕(很快)，如果不能下载插件的话可以本地安装

<!-- more -->

```sh
./logstash-plugin install xx/x-pack-version.zip
```
启动elasticsearch之后，到bin目录执行下面命令初始化密码:
```sh
./bin/x-pack/setup-passwords interactive  //ps:自定义是密码
./bin/x-pack/setup-passwords auto   //ps:随机设置密码
```
浏览器输入http://ip:9200  的时候会提示输入用户名和密码
<img src="/image/elk/es-login.png">

## logstash安装x-pack
切换到logstash的bin目录执行
```sh
./logstash-plugin install x-pack
```
稍等片刻安装完成，此时重启logstash你会发现报错了.....
报错1
```
but got an error. {:url=>"http://logstash_system:xxxxxx@localhost:9200/
```

在logstash.yml中添加
```yml
xpack.monitoring.elasticsearch.url: http://ip:9200
```

报错2
```log
[WARN ][logstash.licensechecker.licensereader] Attempted to resurrect connection to dead ES instance, but got an error. {:url=>"http://localhost:9200/", :error_type=>LogStash::Outputs::ElasticSearch::HttpClient::Pool::HostUnreachableError, :error=>"Elasticsearch Unreachable: [http://localhost:9200/][Manticore::SocketException] Connection refused (Connection refused)"}
```
参考：
https://discuss.elastic.co/t/logstash-with-x-pack/90230
https://www.cnblogs.com/liang1101/p/8509978.html
也可以关闭xpack的monitoring检测
在logstash.yml中添加
```yml
xpack.monitoring.enabled: false
```
在logstash加载的配置文件配置es的用户名和密码:使用es用户的账号，其他账号由于权限问题会导致各种报错
```conf
  elasticsearch{
        hosts => ["ip:9200"]
        index => "xxx-%{+YYYY.MM.dd}"
        user => "esUser"
        password => "espasswd"
      }
```


## kibana安装xpack
切换到kibana的bin目录
```sh
./kibana-plugin install xpack
```
这个过程很慢，可以先去做其他的事情了
修改kibana配置
```sh
vi config/kibana.yml
```
在kibana配置文件添加
```yml
xpack.reporting.encryptionKey: "xxxxxxxxx"
xpack.security.encryptionKey: "something_at_least_32_characters"
elasticsearch.username: "esUser"
elasticsearch.password: "espasswd"
```
启动kibana，访问 http://ip:5601 ,输入用户名密码登录
<img src='/image/elk/kibana.png'>

## 破解-pack（支持购买正版）
安装好x-pack后试用期只有一个月，可以在这个地址申请免费一年license：https://license.elastic.co/registration。  
如果你想要一个长期的证书，看下面的步骤（本人使用idea编译，也可以直接编写Java文件替换）
+ 1.新建LicenseVerifier.java 和 XPackBuild。包名和源文件保持一致（可以解压x-pack-core-xxx.jar查看文件）
+ 2.将`elasticsearch-xxx/plugins/x-pack` 和 `elasticsearch-xxx/lib`文件拷贝到本地项目中
+ 3.编写LicenseVerifier.java 
```java
package org.elasticsearch.license;

public class LicenseVerifier {
    public static boolean verifyLicense(final License license, final byte[] encryptedPublicKeyData) {
        return true;
    }

    public static boolean verifyLicense(final License license) {
        return true;
    }
}
```
+ 4.编写XPackBuild.java
```java
package org.elasticsearch.xpack.core;

import io.netty.util.SuppressForbidden;
import org.elasticsearch.common.io.PathUtils;
import java.net.URISyntaxException;
import java.net.URL;
import java.nio.file.Path;

public class XPackBuild {
    public static final XPackBuild CURRENT;
    private String shortHash;
    private String date;

    @SuppressForbidden(reason = "looks up path of xpack.jar directly")
    static Path getElasticsearchCodebase() {
        final URL url = XPackBuild.class.getProtectionDomain().getCodeSource().getLocation();
        try {
            return PathUtils.get(url.toURI());
        } catch (URISyntaxException bogus) {
            throw new RuntimeException(bogus);
        }
    }

    XPackBuild(final String shortHash, final String date) {
        this.shortHash = shortHash;
        this.date = date;
    }

    public String shortHash() {
        return this.shortHash;
    }

    public String date() {
        return this.date;
    }

    static {
        final Path path = getElasticsearchCodebase();
        String shortHash = null;
        String date = null;
        Label_0157:
        {
            shortHash = "Unknown";
            date = "Unknown";
        }
        CURRENT = new XPackBuild(shortHash, date);
    }
}
```
编译文件后，将生成的class替换到解压的x-pack-core-xxx.jar文件夹中，停止重新生成jar包丢到服务器对应的位置
```sh
jar -cvf x-pack-core-xxx.jar ./*
```
停止es，重启  

我编译好的jar包在这,可以直接使用
<a href="/raw/x-pack-core-6.2.4.jar">x-pack-core-6.2.4.jar</a>  

如果碰到这个问题
`Cannot install a [PLATINUM] license unless TLS is configured or security is disabled`
或者
`Please set [xpack.security.transport.ssl.enabled] to [true] or disable security by setting [xpack.security.enabled] to [false].`
可以在`elasticsearch.yml`文件中添加下面的配置
```yml
# xpack.security.enabled: false和xpack.security.transport.ssl.enabled: true都可以解决这个问题，任选一个
#xpack.security.enabled: false 
xpack.security.transport.ssl.enabled: true
```
修改上面下载的证书json文件
```json
{
        "license": {
                "uid": "你的uuid", 
                "type": "platinum", # 修改授权为白金版本
                "issue_date_in_millis": 1526860800000,
                "expiry_date_in_millis": 2524579200999, #修改到期时间
                "max_nodes": 100, # 修改最大节点数
                "issued_to": "xxxx", #你申请时填写的，不需要动
                "issuer": "xxx", #你申请时填写的，不需要动
                "signature": "xxxxxxaasa", #原文件中的签名，不需要动
                "start_date_in_millis": 1526860800000
        }
}
```

导入证书
```sh
curl -u elastic:elastic -XPUT 'http://es-ip:port/_xpack/license' -H "Content-Type: application/json" -d @license.json
``` 
返回如下数据，说明激活成功
```json
{"acknowledged":true,"license_status":"valid"}[
```
将elk都重启后，打开kibana可以看到有效日期变化了。
<img src='/image/elk/license.png'>