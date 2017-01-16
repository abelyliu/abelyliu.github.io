---
title: ExtJs下拉框选择空
date: 2016-12-22 08:25:01
tags: Ext JS
category: JavaScript
---

在Ext中，我们使用下拉框时，如果选择一个选项之后，很难撤销选中，尤其在不可编辑的下拉框中。此时我们提供一个默认的空选项就很有用处，如下所示：

![](/images/50.png)
<!--more-->

关于这个问题，网上有很多解法，要么比较复杂如：使用tpl，还需配置css，要么效果不是那么令人满意，不过有一种方法还是比较简单实用的。我们知道combo需要配置一个store，store中的数据就是要显示的数据，只要配置一个“空”数据即可。

## 本地数据
方式一：
```javascript
var states = Ext.create('Ext.data.Store', {
    fields: ['abbr', 'name'],
    data: [
        {abbr: '\u00a0', name: 'null'},
        {abbr: "AL", name: "Alabama"},
        {abbr: "AK", name: "Alaska"},
        {abbr: "AZ", name: "Arizona"},
    ]
});
```
上述数据中abbr是combo显示的内容，而name着是选择的值，`\u00a0`对应的值可以根据自己的需求进行修改,我的逻辑是当选中它时，值为'null'。

方式二：
```javascript
states.add({abbr: '\u00a0', name: 'null'});
```
在combo中需要配置数据加载方式为本地：
```javascript
{
    xtype: 'combo',
    fieldLabel: 'Start date',
    store: states,
    id: 'test',
    editable: false,
    valueField: 'name',
    //必须设置queryMode，否则不会添加null
    queryMode: 'local',
    displayField: 'abbr',
}
```

## 远程数据
服务器返回的数据内容格式如下：

![](/images/51.png)

thisSite是我要显示在combo中的值，我在success回调函数中加入了下面一句话
```javascript
action.result.importDeviceInfo.thisSite.unshift({"name":"\u00a0","server_id":"null"});
```
然后在store定义如下：
```javascript
var states = Ext.create('Ext.data.Store', {
   fields: ['name', 'server_id'],
   data: action.result.importDeviceInfo.thisSite,
   sorters: [{
       property: 'name',
       direction: 'desc'
   }],
});
```
因为我在store中对数据进行了倒序排序，所以一开始使用unshift方法和push方法差别不大。
