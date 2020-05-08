---
title: this的指向
date: 2020-03-12 21:01:30
tags:
---

JavaScript是一门词法作用域的语言，为了实现动态的能力，JavaScript引入了this。词法作用域又叫静态作用域，与之相对的是动态作用域，而动态作用域则是指根据函数调用的位置来确定变量作用范围这种规则。

而JavaScript实现动态作用域的方法就是通过this，this在函数体内部，指向函数当前的运行环境。

因为this会有多种出现方式，我们归纳为四种，优先级从高到低分别为：

1. 构造绑定，指的是函数作为构造函数，通过 new foo()这种方式创建的对象，在这种情况下，foo函数中的所有this都指向创建出的新的对象。
2. 显式绑定，指的是通过call、apply、bind这种很显然的方法来对函数foo进行this的绑定，此时，foo中的所有this指向都会指向call、apply、bind传入的第一个参数中的对象。
3. 隐式绑定，指的是通过obj.foo()这种隐式的方法来调用函数时，foo中的this都指向obj这个对象。
4. 默认绑定，在默认情况下，this指向全局对象。其实这种情况可以看作是第三种情况隐式绑定的特殊情况，因为在浏览器环境下，全局环境就是window，而window通常会被省略(window.foo() 省略变成foo())。

如果以上几种情况出现了冲突，那么按照优先级从高到低的顺序决定this的指向。



## 参考资料

1. [Javascript 对象基础-MDN](https://developer.mozilla.org/zh-CN/docs/Learn/JavaScript/Objects/Basics)
2. [JavaScript的this原理-阮一峰](https://www.ruanyifeng.com/blog/2018/06/javascript-this.html)
3. [掌握JS中的指向只需要记忆五大原则](https://juejin.im/entry/5a20d18af265da43062a9031)
4. [JavaScript深入之从ECMAScript解读this](https://github.com/mqyqingfeng/Blog/issues/7)
5. [ECMAScript规范中文版](http://yanhaijing.com/es5/#80)

