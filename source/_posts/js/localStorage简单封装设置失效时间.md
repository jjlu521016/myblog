---
title: localStorage简单封装设置失效时间
toc: true
date: 2018-04-24 09:30:09
tags:
    - localStorage
---
### 1.概述：
在客户端存储数据
HTML5 提供了两种在客户端存储数据的新方法：
+ localStorage - 没有时间限制的数据存储
+ sessionStorage - 针对一个 session 的数据存储

localStorage和cookie 的区别不详细对比，但是localStorage存储数据的时候有一点需要我们注意的。
<!-- more -->
### 2.场景：
我们前端需要调用第三方api异步获取我们的数据A（短时间内数据都一样）,然后我们拿到数据A来进行其他操作。试想下从获取数据A再到用数据A获取我么想要的最终结果，这段时间对用户来说是很漫长了！ 
其实我们可以使用
+ Cookie来存储数据，但是Cookie存储的数据有限制。
+ 使用localStorage能满足存储数据的条件，但是它却没有失效时间。
那我们改怎么优化这种场景呢？

综上所述，Cookie已经无法满足我们的要求了，那么我们就从localStorage入手吧。既然localStorage没有失效时间，我们就封装下使其满足我们的需求。

### 3.实现：
下面的代码仅供参考：
```javascript
var storageUtil = {
    /**
     * @param key
     * @param data
     * @param time 失效时间（秒）,默认一周
     * @returns {boolean}
     */
    put: function (key, data, time) {
        try {
            if (!localStorage) {
                return false;
            }
            if (!time || isNaN(time)) {
                time = 60*60*24*7;
            }
            var cacheExpireDate = (new Date() - 1) + time * 1000;
            var cacheVal = {val: data, exp: cacheExpireDate};
            localStorage.setItem(key, JSON.stringify(cacheVal));//存入缓存值
        } catch (e) {
        }
    },
    get: function (key) {
        try {
            if (!localStorage) {
                return false;
            }
            var cacheVal = localStorage.getItem(key);
            var result = JSON.parse(cacheVal);
            var now = new Date() - 1;
            //缓存不存在
            if (!result) {
                return null;
            }
            //缓存过期
            if (now > result.exp) {
                this.remove(key);
                return "";
            }
            return result.val;
        } catch (e) {
            this.remove(key);
            return null;
        }
    },
    remove: function (key) {
        if (!localStorage) {
            return false;
        }
        localStorage.removeItem(key);
    }
}
```
### 4.使用实例
```javaScript
storageUtil.put("aa",JSON.stringify(data));

var cacheData = storageUtil.get("aa");
if(cacheData !== null || cacheData !== ''){
    
}else{

}
```