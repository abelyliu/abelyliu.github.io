---
title: Ext Js返回Json串出错
date: 2016-11-28 10:40:00
tags: Ext JS
category: JavaScript
---
今天从后台返回Json字符串时，Firefox控制台一直出现`Ext.Error: You're trying to decode an invalid JSON String: <pre>*</pre>`，Chrome则是`ext-all.js:21 Uncaught Ext.JSON.decode(): You're trying to decode an invalid JSON String: <pre style="word-wrap: break-word; white-space: pre-wrap;">*</pre>`，中间的*是我要返回的Json字符串，上网查询发现如下解决方法。
<!--more-->

本人使用Struts2返回的Json串，只需要将返回头application/json改为text/html即可。Struts2配置文件修改如下：
```xml
 <result type="json">
    <param name="contentType">text/html</param>
    <param name="root">importDeviceInfo</param>
</result>
```
[参考链接](http://yedward.net/?id=374)