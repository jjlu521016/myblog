---
title: Jquery中.attr和.data的区别
toc: true
date: 2018-03-16 17:10:37
tags:
---
$.attr()和$.data()本质上属于 `DOM属性` 和 `Jquery对象属性` 的区别：

+ $.attr()每次都从DOM元素中取属性的值。
+ $.attr('data-xxx', 'xxxxx')会将字符串'xxxx'塞到标签的'data-xxx'属性中。
<!-- more -->
+ $.data('xxx')是从 Jquery对象中取值，由于对象属性值保存在内存中，因此可能和视图里的属性值不一致的情况。
+ $.data('xxx', 'xxxx')会将字符串'xxxx'塞到 Jquery对象 的'xxx'属性中，而不是塞到视图标签的data-xxx属性中。

所以$.attr()和$.data()应避免混合用
+ 通过$.attr()来进行set属性，然后通过$.data()进行get属性值；
+ 通过$.data()来进行set属性，然后通过$.attr()进行get属性值。
同时从性能的角度来说，建议使用$.data()来进行set和get操作，因为它仅仅修改的 Jquey对象 的属性值，不会引起额外的DOM操作。 