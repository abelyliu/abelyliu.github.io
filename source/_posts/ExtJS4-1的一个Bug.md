---
title: Ext JS 4.1的一个Bug
date: 2016-12-20T16:03:39.000Z
tags: Ext JS
category: JavaScript
---
在Ext4.1版本中使用遮罩配置选项时无法产生效果，遮罩效果是在加载数据时提示正在加载数据的有效方式，二如果想使用此效果，我们可以添加如下代码。
 <!-- more -->

添加如下代码：
```javascript
Ext.override(Ext.view.AbstractView, {
    onRender: function() {
        var theObj = this;
        theObj.callOverridden;
        if (theObj.mask && Ext.isObject(me.store)) {
            theObj.setMaskBind(me.store);
        }
    }
})
```

参考链接为[这里](https://www.sencha.com/forum/showthread.php?198680-ext-4.1-Load-mask-on-grid-gone)。
