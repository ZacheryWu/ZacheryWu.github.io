---
title: js的事件循环
date: 2020-03-31 21:21:14
tags:
---

 js主要有浏览器和node这两个执行环境，虽说node采用了V8作为js的解析引擎，但使用了libuv来进行I/O处理。因此js在浏览器和node端的事件循环不一样，也导致相同代码的执行结果可能不一样。

### 浏览器端

浏览器端的事件循环分为三块：

1. 执行栈
2. 宏任务队列(macro-task)
3. 微任务队列(micro-task)

![img](D:\hexo\blog\source\images\16740fa4cd9c6937)

#### 执行栈

执行栈，其实就是执行上下文栈，之前在作用域链的文章中也曾经介绍过。执行上下文栈是用来保存执行上下文的一个栈结构，先进后出。在一段js代码开始执行前，会先把全局的执行上下文压入栈中，然后根据栈顶的执行上下文来执行代码，在执行的过程中，如果遇到了函数，也会创建函数的执行上下文，并且压入栈中，当这个函数执行完毕，就会把这个函数对应的执行上下文弹出，当整段代码执行完毕，也会把全局的执行上下文从栈中弹出，代表着代码执行的结束。

而在执行栈中的代码全部执行完毕后，就回去宏任务队列和微任务队列中取出任务执行。

#### 宏任务队列

除了同步执行的任务，还有一些异步执行的任务会被放到宏任务队列中，如：

- setTimeout
- setInterval
- xhr
- ......

这些需要异步执行的任务会在满足某个条件后被放入宏任务队列中，等待着执行栈从队列里拿任务执行。执行栈在当前的任务全部执行完毕后，宏任务队列会出队一个任务供执行栈来执行。

#### 微任务队列

像Promise、generate这种就属于微任务队列，会被js引擎放到为任务队列中去。

当执行栈在当前的任务执行完毕后，会先把微任务队列中的全部任务按照顺序执行掉，然后再去执行宏任务队列中出队的那个任务。

#### 浏览器异步任务执行顺序总结

首先执行同步代码，当同步任务执行完毕，会先把微任务队列清空，然后再从宏任务队列中出队一个任务来执行。(挺简单的哦)



### node端

node端的事件循环就复杂了很多，先来简单介绍一下node.js的运行机制：

1. V8引擎解析JavaScript脚本
2. 解析后的代码调用Node API
3. libuv库负责Node API的执行。把不同的任务分配给不同的线程，形成一个事件循环，以异步的方式将任务的执行结果返回给V8引擎。
4. V8引擎将结果返回给用户

#### node 的事件循环

libuv引擎中的事件循环分为六个阶段，按照下图的顺序反复执行。每到一个阶段都会从对应的回调队列中去除函数执行。当队列为空或者执行的回调函数数量达到了阈值就会进入下一个阶段的执行。

![img](D:\hexo\blog\source\images\1)

简单描述以下就是：

1. 外部输入数据
2. 轮询阶段(poll)，用来获取新的I/O事件
3. 检查阶段(check)，执行setImmediate的回调
4. 关闭事件回调(close callbacks)阶段，执行socke的close事件
5. 定时器检测阶段(timers)，执行定时器相关的回调
6. I/O事件回调阶段(I/O callbacks)，执行i/O回调
7. 闲置阶段(idle，prepare)
8. 回到第二步

#### node中的宏任务和微任务

微任务： process.nextTick、promise等

宏任务：setTimeout、setImmediate等

在node中，微任务会穿插在每个阶段执行之后执行。而且因为版本的不同，也会不太一样，

- 在node11以后的版本中，会在执行一个宏任务之后，就执行微任务。
- 在node10及以前的版本，会在整个阶段执行完毕之后再执行微任务。

所以下面的代码在不同的版本中，会出现不同的结果

```js
setTimeout(()=>{
    console.log('timer1')
    Promise.resolve().then(function() {
        console.log('promise1')
    })
}, 0)
setTimeout(()=>{
    console.log('timer2')
    Promise.resolve().then(function() {
        console.log('promise2')
    })
}, 0)
node10:timer1=>timer2=>promise1=>promise2
node11:timer1=>promise1=>timer2=>promise2
```

[这篇文章](https://juejin.im/post/5c337ae06fb9a049bc4cd218)的 这两张图十分形象

node11: 

![img](D:\hexo\blog\source\images\node11.gif)

node10:

![img](D:\hexo\blog\source\images\node10.gif)

### 总结

浏览器和node11在异步的处理上达到了一样的结果，而在node11之前的版本则会先处理整个宏观队列，导致不同的结果。


### 参考资料
[浏览器与Node的事件循环(Event Loop)有何区别?](https://juejin.im/post/5c337ae06fb9a049bc4cd218)
[[译] 理解 JavaScript 中的执行上下文和执行栈](https://juejin.im/post/5ba32171f265da0ab719a6d7)
掘金小册-前端面试之道-Event Loop
