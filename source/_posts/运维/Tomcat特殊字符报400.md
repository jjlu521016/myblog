---
title: Tomcat特殊字符报400（The valid characters are defined in RFC 7230 and RFC 3986）
date: 2018-03-07 17:02:34
tags: 运维
---


## <center>Tomcat特殊字符报400（The valid characters are defined in RFC 7230 and RFC 3986）</center>

* 报错信息如下：
```log
org.apache.coyote.http11.Http11Processor.service Error parsing HTTP request header
 Note: further occurrences of HTTP header parsing errors will be logged at DEBUG level.
 java.lang.IllegalArgumentException: Invalid character found in the request target. The valid characters are defined in RFC 7230 and RFC 3986
```
Tomcat在 7.0.73, 8.0.39, 8.5.7 等版本后(详情：https://stackoverflow.com/questions/41053653/tomcat-8-is-not-able-to-handle-get-request-with-in-query-parameters/44005213#44005213)，添加了对于http头的验证,就是添加了些规则去限制HTTP头的规范性(详情：http://www.jianshu.com/p/1c870461fa41)

* 根据rfc规范（RFC 3986规范定义了Url中只允许包含英文字母（a-zA-Z）、数字（0-9）、-_.~4个特殊字符以及所有保留字符(RFC3986中指定了以下字符为保留字符：! * ’ ( ) ; : @ & = + $ , / ? # [ ])）。
* url中不允许有 |，{，}等特殊字符，但在实际生产中还是有些url有可能携带有这些字符，特别是|还是较为常见的。在tomcat升级到7以后，对url字符的检查都变严格了，如果出现这类字符，tomcat将直接返回400状态码。

 
* 修改tomcat的配置，在catalina.properties添加下面的配置，重启服务器即可。
```sh 
tomcat.util.http.parser.HttpParser.requestTargetAllow=|{}
```
* tomcat官方说明
http://tomcat.apache.org/tomcat-7.0-doc/config/systemprops.html;
https://tomcat.apache.org/tomcat-8.5-doc/config/systemprops.html

* 源码片段如下：
```java
    if (IS_CONTROL[i] || i > 127 ||
                    i == ' ' || i == '\"' || i == '#' || i == '<' || i == '>' || i == '\\' ||
                    i == '^' || i == '`'  || i == '{' || i == '|' || i == '}') {
                IS_NOT_REQUEST_TARGET[i] = true;  // reject the character!
```