---
title: JavaScript之this与闭包
date: 2016-08-16T11:44:59.000Z
category: JavaScript
tags: JavaScript
---

在JavaScript的闭包学习过程中，发现在闭包中使用this会出现很奇怪的现象，但并没有想明白，直到看到了你不知道的JS这本书(不过我更喜欢你不懂JS的译法)。
<!--more-->
关于this的问题也不说了，这只简单讨论一下this的绑定的几种方式。其实书中有关于this更细致的讨论，如有兴趣，自行阅读You-Dont-Know-JS之[this & Object Prototypes](https://github.com/getify/You-Dont-Know-JS)。

### 默认绑定

```javascript
  function foo(){
    console.log(this.a);
  }
  var a = 2;
  foo();
```

上述的情形就是this默认绑定的方式，在默认绑定的情况下this等同于全局对象，在浏览器下就是window。在严格模式下，this被禁止使用默认绑定。

```javascript
  function foo(){
    "use strict";
    console.log(this.a);
  }

  var a = 2;
  foo();    //typeError:this is undefined
```

### 隐式绑定

```javascript
var a = 12;
function foo(){
  console.log(this.a);
}

var obj1 = {
  a : 42,
  foo : foo
}

var obj2 = {
  a : 2,
  foo : foo,
  obj1 : obj1
}

var obj3 = obj1.foo;

foo();            //12  默认绑定
obj1.foo();   //42    隐式绑定
obj2.foo();   //2     隐式绑定
obj2.obj1.foo();  //42  隐式绑定
obj3();         // 12  默认绑定
```

上面的方式是隐式绑定，foo被隐式的绑定到obj1,obj2中，从某种角度来说，obj1和obj2对象中持有foo，并且是通过obj1或obj2调用的。大家发现没有，最后一种是默认绑定，obj3实质上就是等于foo，也就是第一种调用方式，值得注意。再看个例子：

```javascript
  var a = 12;
  function foo(){
    console.log(this.a);
  }

  var obj1 = {
    a : 42,
    foo : foo
  }

  function doFoo(fn) {
    fn();
  }

  doFoo(obj1.foo);  //12
```

### 显示绑定

```javascript
  function foo(){
    console.log(this.a);
  }

  var obj1 = {
    a:42
  }

  foo.call(obj1);   //42
```

当我们使用call和apply调用函数时，函数中的this会被绑定到call或apply的第一个参数上。此例中foo中this被显式绑定到obj1上。

### new绑定

```javascript
  function foo(a) {
    this.a = a;
  }

  var bar = new foo(2);
  console.log(bar.a);   //   2
```
我们发现this被绑定到bar上去了，这就是new绑定。

### 优先级

默认绑定<隐式绑定<显式绑定<new绑定。

如果大家对证明过程感兴趣也可以参考书中的内容。
