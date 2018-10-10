---
title: cordova不能拨打电话号码的解决办法
toc: true
date: 2015-08-02 13:53:49
tags:
    - cordova
---
在html里面直接使用不能拨打电话的解决方法如下：

1. 下载cordova的访问白名单插件 ：

命令：
```sh
cordova plugins add cordova-plugin-whitelist
``
2. 在config.xml里面添加 如下配置：
```xml
<content src = "index.html"/>

<plugin name = "cordova-plugin-whitelist" version="1" />

<access orgin = "*" />

<allow-intent href = "http://*/*" />

<allow-intent href = "https://*/*" />

<allow-intent href = "tel:*" />

<allow-intent href = "sms:*" />

<allow-intent href = "mailto:*" />

<allow-intent href = "geo:*" />

<platform name = "android">

<allow-intent href = "market:*" />

</platform>

<platform name = "ios" >

<allow-intent href = "itms:*" />

<allow-intent href = "itms-apps:*" />

<platform>
```