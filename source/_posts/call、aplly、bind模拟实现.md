---
title: call、aplly、bind模拟实现
date: 2020-03-17 17:32:58
tags:
---

 除了点操作符也就是隐式绑定外，还有几种方式可以影响this的指向，下面来模拟实现以下这几种方式

## call、apply、bind

### call的模拟实现

	call() 方法使用一个指定的this值和单独给出的零个、一个或多个参数来调用一个函数。	-MDN

根据上面这句话我们可以知道，call方法主要有两个功能：

1. 给函数指定this值
2. 调用函数

根据上面两个条件，写出下面的代码。

```js
Function.prototype.call = function(obj, ...args) {
	obj = obj || window;
	obj.fn = this;
	let result = obj.fn(...args);
	delete obj.fn;
	return result;
}

function printThis() {
	console.log(this);
}
let o = {str: '一句话'};
printThis.call(o);//Object o
```

然而我们用到了ES6的解构赋值，而call在ES3就已经出现了，所以下面不使用ES6的特性实现一遍：

```js
Function.prototype.call = function(obj) {
	obj = obj || window;
	obj.fn = this;
	var args = [];
	for(var i = 1; i < arguments.length; i++) {
		args.push('arguments[' + i + ']');
	}
	var result = eval('obj.fn('+ args + ')');
	return result;
}
function printThis(a, b) {
	console.log(this);
	console.log(a + b)
}
let o = {str: '一句话'};
printThis.call(o, 1, 2);
```

不使用解构赋值的情况下，选择使用eval来实现函数参数的传递。

### apply的模拟实现

	apply方法调用一个具有给定this值得函数，以及一个数组作为函数的参数。	-MDN

apply方法和call几乎一样，除了传参的方式，在这里只实现一个ES6版本的模拟一下：

```js
Function.prototype.apply = function(obj, args) {
	obj = obj || window;
	obj.fn = this;
	let result = obj.fn(...args);
	delete obj.fn;
	return result;
}
function printThis(a, b) {
	console.log(this);
	console.log(a + b)
}
let o = {str: '一句话'};
printThis.apply(o, [1, 2]);
```

其实上面的方式中都在对象o上添加了一个fn函数，这可能会覆盖掉原本对象o中的方法，并不是一个完美的实现。

### bind的模拟实现

```
bind()方法创建一个新的函数，在bind()被调用时，这个新函数的this被指定bind()的第一个参数，而其余参数作为新函数的参数，供调用时使用。	-MDN
```

根据上面这段话我们可以知道，bind有三个功能：

1. 创建一个新的函数
2. 给这个新的函数绑定this
3. 给新函数传递参数

根据上面的功能，我们实现的bind如下：

```js
Function.prototype.bind = function(obj, ...args) {
	obj = obj || window;
	var self = this;
	return function() {
		return self.apply(obj, args);
	}
}
var foo = {value : 1};
function bar(name) {
	console.log(this.value);
	console.log(name);
} 
let fooBar = bar.bind(foo, 'Goodman');
fooBar();
//1
//Goodman
```

可以看到bind的基本功能都已经实现了（使用了我们上面实现的apply）。然而bind还有一些细节需要去雕琢一下：不仅可以在bind调用时传递参数，也可以在返回的新函数调用时传递参数，但是bind绑定时的参数优先。

下面是对于这条规则的实现：

```js
Function.prototype.bind = function(obj, ...args0) {
	obj = obj || window;
	var self = this;
	return function(...args1) {
		return self.apply(obj, [...args0, ...args1]);
	}
}
```

## new 与 call、apply、bind的恩怨情仇

在之前学习this的指向时，了解到，new 和 call、apply、bind都可以改变this的指向，而当call、apply、bind遇到new时则会失效，以new为准。那么之前模拟实现的这些方法，是否符合这条规则呢，下面就来分析一下。

### new 的模拟实现

在分析那些规则之前，先来了解下new。

```js
new 运算符创建一个用户定义的对象类型的实例或具有构造函数的内置对象的实例。会进行以下操作：
1. 创建一个Javascript的空对象。
2. 链接该对象到另一个对象，即设置对象的原型为构造函数的prototype指向的对象。
3. 将该对象作为this的上下文。
4. 如果函数没有返回对象，则返回创建的这个对象。
```

模拟实现以下new，new是操作符，所以不能像之前的函数一样直接覆盖，下面用函数来模拟new的实现：

```js
function newFactory(fn, ...args) {
	var obj = Object.create(null);
	obj.__proto__ = fn.prototype;
	var ret = fn.apply(obj, args);
	return typeof ret === 'object' ? ret : obj;
}
```

实际上，这样的写法是不行的，如果使用下面的代码进行测试就会报错：

```js

function Otaku (name, age) {
    this.name = name;
    this.age = age;

    this.habit = 'Games';
}

Otaku.prototype.strength = 60;

Otaku.prototype.sayYourName = function () {
    console.log('I am ' + this.name);
}

var person = newFactory(Otaku, 'Kevin', '18')

console.log(person.name) // Kevin
console.log(person.habit) // Games
console.log(person.strength) // undefined

person.sayYourName(); // error: person.sayYourName is not a function
```

报错的原因是Object.create(null)创建出来的对象，是没有原型链的，外在体现就是创建出的对象没有`__proto__`这个属性，所以也无法通过`__proto__`这个属性来修改对象的原型链。将代码修改为下面的就可以正常使用了：

```js
function newFactory(fn, ...args) {
	var obj = {};
	obj.__proto__ = fn.prototype;
	var ret = fn.apply(obj, args);
	return typeof ret === 'object' ? ret : obj;
}
```

下面来分析以下new操作符和其他几个方法的关系

### new 和call、apply

由于new操作符的优先级比方法执行更高，所以在两者同时出现时，会先执行new操作符，如下面的代码：

```js
function test() {
    this.value = 123
    console.log(this)
}

let applyObj = test.call({value: 'apply'});
let newObj = new test().call({value: 'apply'});//Uncaught TypeError: (intermediate value).call is not a function
```

最终的结果是报错，但是确实是new的优先级更高。而new执行完之后会返回一个对象，继续执行代码，这个对象上如果没有call方法就会报错。

apply同理。

### new和bind

new和bind的比较就不能从优先级的角度来了，在bind的模拟视线中，bind绑定过的方法实际上是通过闭包强行改变了this的指向，所以，我们为了能够符合这条规则，需要对bind方法进行一些修改：

```js
Function.prototype.bind = function(obj, ...args0) {
	obj = obj || window;
	var self = this;
	var fBound = function (...args1) {
		return self.apply(this instanceof fBound ? this : obj, [...args0, ...args1]);
	}
	return fBound;
}
```

这时函数里的this确实指向了新创建的对象。但是还有一个问题，那就是创建的对象的原型是指向fBound的prototype的，而不是我们绑定前的那个函数的prototype。所以再修改一下:

```js
Function.prototype.bind = function(obj, ...args0) {
	obj = obj || window;
	var self = this;
	var fBound = function (...args1) {
		return self.apply(this instanceof fBound ? this : obj, [...args0, ...args1]);
	}
	fBound.prototype = this.prototype;
	return fBound;
}
```



## 总结

之前在工作中用到的call、apply、new这些其实并不多，所以对这块的东西也一直云里雾里的，这次模拟实现了一下发现其实并不难，虽然还有些小瑕疵，但是极大的加深了对this使用的理解。



### 参考资料

[JavaScript深入之call和apply的模拟实现](https://github.com/mqyqingfeng/Blog/issues/11)

[JavaScript深入之bind的模拟实现](https://github.com/mqyqingfeng/Blog/issues/12)

[JavaScript深入之new的模拟实现](https://github.com/mqyqingfeng/Blog/issues/13)

[new 运算符 - MDN](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Operators/new)

[Function.prototype.call - MDN](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Function/call)

[Function.prototype.apply - MDN](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Function/apply)

[Function.prototype.bind - MDN](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Function/bind)

