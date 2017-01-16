---
title: ExtJs水平居中组件
date: 2016-12-09 08:38:06
tags: Ext JS
category: JavaScript
---
需要做一个两个下拉框居中显示的效果，而且窗口大小改变时还是居中显示。简单的查阅了一下文档，效果图如下：
![](/images/47.png)

<!--more-->

要想实现上述效果需要实现下面三个内容：
1. 两个日历选择控件要水平显示
2. 调整组件间的间距
3. 改变窗口大小时，重新绘制界面

## hbox布局
> A layout that arranges items horizontally across a Container. This layout optionally divides available horizontal space between child items containing a numeric flex configuration.

hbox布局是让容器内的子元素水平按顺序排列,hbox有两个常见的配置项align和pack。align代表子元素在垂直方向的排列方式，pack代表在水平方向的排列方式。

align配置项|作用
:---:|:---:
top|顶部对齐，从容器的top开始垂直对齐
middle|居中对齐，从容器的middle开始垂直对齐
stretch|拉伸子项以填充容器的垂直高度
stretchmax|拿最高的那个子项拉伸，适应容器的垂直高度

----

pack配置项|作用
:---:|:---:
start|居左，从左边开始排列内容
center|居中，从容器的mid-width开始停靠内容
end|居右，从容器的右边开始停靠内容那个

更多的细节特性可以参考[这篇博客](http://blog.csdn.net/zhangxin09/article/details/5631640)。

代码如下：
```javascript
var panel =  Ext.create('Ext.panel.Panel', {
   bodyPadding: 5,  // Don't want content to crunch against the borders
   border:0,
   layout: {
       type: 'hbox',
       align: 'middle',
       pack: 'center',
   },
   items: [{
       xtype: 'combo',
       fieldLabel: 'Start date',
       margin: '0 50 0 50'
   },{
       xtype: 'combo',
       fieldLabel: 'End date'
   }],
   renderTo: Ext.getBody()
});
```

我们可以发现在第一个子项中有一句`margin: '0 50 0 50'`，这个就是来调节和下一个子项之间的间距，也可在panel中加入默认的间距`defaults:{margins: '0 50 0 50'}`。

## 自动居中
如果仅仅依靠上面的代码，仅仅在第一次打开页面或者刷新时组件才会居中，如果我们修改了浏览器窗口的大小，组件是不会改变，而且不会出现滚动条。其中的一个实现办法是监听浏览器大小改变事件，然后重新绘制组件。
```javascript
Ext.EventManager.onWindowResize(function(w, h){
    panel.doComponentLayout();
});
```
我们还有另一个办法，就是使用panel的监听机制，panel有个resize事件，同样可以监听到浏览器窗口的改变，因为会影响到panel的大小。
```javascript
Ext.create('Ext.panel.Panel', {
    style: 'margin:0 auto;border-width:0 0 0 0;',
    width: '100%',
    id: "<portlet:namespace />PortletPanel",
    items: [
        plant_site, tabPanel
    ],
    listeners: {
        resize: function (w, h) {
            this.doComponentLayout();
        }
    },
    renderTo: '<portlet:namespace />MesMigrationPanel'
});
```
