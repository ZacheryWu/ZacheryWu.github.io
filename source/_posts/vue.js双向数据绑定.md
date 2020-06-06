---
title: vue.js双向数据绑定
date: 2019-03-06 16:45:01
tags: Vue
---

## 前言

`Vue.js`有很多有用的特性，其中最显著的就是双向绑定，第一次接触到`Vue.js`的时候，就被这种简单方便的特性吸引了。现在自己也尝试着用`Vue.js`中使用的`Object.defineProperty`来实现双向绑定。

最终的效果是：

![vueSlef_result.gif](https://i.loli.net/2019/08/27/Qs3xhNyMtFgkdaD.gif)


```html
	<div id="app">
		<input type="text" v-model='text'>
		{{text}}
	</div>
	<script src="./Vue.js"></script>
	<script>
		let vm = new Vue({
			el: 'app',
			data: {
				text: 'Hello Vue!'
			}
		});
	</script>
```



## 分步实现

看起来只是一个简单的双向绑定的功能，其实可以分为三个小的步骤：

- 解析html，把html中`{ {text} }`替换，并且给input赋值。
- 把input的改变同步到data中的对应数据上。
- 把data中数据的改变同步到html中绑定的数据上。

### 解析html

由于只是实现一个简单的双向绑定，所以只对`{ { } }`和`v-model` 进行解析，并且只考虑了只有一层dom节点的情况。距离实际使用还差得远呢。（这里hexo对这个双大括号无法解析，只能加了几个空格😔）

```js
class Vue {
	constructor(options) {
		this.el = options.el;
		this.data = options.data;
		let root = document.getElementById(this.el);
		let frag = this.node2Fragment(root);
		root.appendChild(frag);
	}
	/**
	 * 把dom转化为documentFragment，可以带来更好的性能。
	 */
	node2Fragment(node) {
		let retFrag = document.createDocumentFragment();
		let child;
		while(child = node.firstChild) {
			this.compile(child);
			retFrag.appendChild(child);
		}
		return retFrag;
	}

	/**
	 * 对dom节点进行解析，把data中的数据替换上去
	 */
	compile(node) {
		let reg = /\{\{(.*)\}\}/;
		if(node.nodeType === 1) {
			let attrs = node.attributes;
			for(let i = 0; i < attrs.length; i++) {
				if(attrs[i].nodeName == 'v-model') {
					node.value = this.data[attrs[i].nodeValue];
				}
			}
		} else if(node.nodeType === 3) {
			if(reg.test(node.nodeValue)) {
				let dataKey = RegExp.$1;
				dataKey = dataKey.trim();
				node.nodeValue = this.data[dataKey];

			}
		}
	}
}
```

学习这部分源码时遇到的问题：

1. `document.createDoumentFragment()`

   通过这个方法可以创建一个指向空documentFragment对象的引用，documentFragment也是DOM节点，但是不是DOM树的一部分，通常用来创建文档片段，将元素附加到文档片段。在把documentFragment添加到DOM树时，文档片段会被它的子元素所替代。

   因为文档片段只存在于内存中而不是DOM树中，所以将子元素插入到文档片段时不会引起页面的回流，因此使用文档片段可以提高页面的性能。

2. `appendChild`

   ```js
   while(child = node.firstChild) {
       this.compile(child);
       retFrag.appendChild(child);
   }
   ```

   看到上面这段代码的时候，很疑惑，因为没有找到循环的变化。查阅资料后发现，appendChild在把参数节点添加过去的同时，也会从原节点上删除这个参数节点。（简单点说就是剪切操作！而不是复制操作。）

3. `node.nodeType `

   用来表示节点的类型，代码中用到了1和三，1表示文本节点，一般是element或者attr中实际的文字；3表示元素节点，比如div、p等等。(并不是带标签的都是3，span和img就是1)

4. `node.attributes`

   这个属性表示node上的所有attr，需要注意的是，node.attributes虽然也有length属性，但是却并不是数组。

### 把input的value变化同步到data中

这一步其实很简单，只需要给input添加一个事件，在input改变的时候同时修改data就可以了。

所以修改一下compile方法：

```js
compile(node) {
    let reg = /\{\{(.*)\}\}/;
    if(node.nodeType === 1) {
        let attrs = node.attributes;
        for(let i = 0; i < attrs.length; i++) {
            if(attrs[i].nodeName == 'v-model') {
                node.addEventListener('input', (e) => {
                    this.data[attrs[i].nodeValue] = e.target.value;
                    console.log(this.data[attrs[i].nodeValue]);
                })
                node.value = this.data[attrs[i].nodeValue];
            }
        }
    } else if(node.nodeType === 3) {
        if(reg.test(node.nodeValue)) {
            let dataKey = RegExp.$1;
            dataKey = dataKey.trim();
            node.nodeValue = this.data[dataKey];

        }
    }
}
```

### 同步data中的数据改动同步到视图

这一步才是实现双向绑定的重点，通过Object.defineProperty 修改data的访问器属性，从而达到对于数据变化的劫持，劫持到之后通知视图进行变化。简单点说就是通过Object.defineProperty实现一个观察者模式。

1. 首先劫持到对data的访问

   ``` js
   constructor(options) {
       this.el = options.el;
       this.data = options.data;
   
       this.observe(this.data);
       let root = document.getElementById(this.el);
       let frag = this.node2Fragment(root);
       root.appendChild(frag);
   }
   
   observe(data) {
       Object.keys(data).forEach((prop) => {
           this.defineReactive(prop, data[prop]);
       })
   }
   
   defineReactive(prop, value) {
       Object.defineProperty(this, prop, {
           get: function() {
               return value;
           },
           set: function(newValue) {
               if(value === newValue) return;
               value = newValue;
           }
       })
   }
   ```

   get可以劫持到对data属性的读操作，set可以劫持到写操作。我们在劫持的同时把data加到了Vue的上，方便我们的访问，所以对于data的访问就可以简写为`this.prop`。（prop是data中的属性）

    

2. 接下来要做的就是收集依赖，并且在数据发生改变时通知依赖变化。

   之前我一直搞不懂是怎么收集依赖的，在这次学习时终于想通了，原来是这么的简单。`如果一个数据a依赖数据b，那么在计算a的值时必然回去读取b的值。`所以在读取数据的时候，也就是触发访问器属性中的get时就可以完成依赖的收集。

   相同的，在数据发生变化时，通过访问器中的set就可以通知到这个数据的watcher（监听器）去同步视图的变化。

   所以我们要做的就是构造一个Dep（主题）和一个Watcher（监听器），然后在set和get的时候添加依赖和触发监听。

   ``` js
   class Watcher {
   	constructor(node, name, vm) {
   		//标记自身，方便Dep收集依赖
   		Dep.target = this;
   		this.node = node;
   		this.name = name;
   		this.vm = vm;
   		this.getVal();
   		//收集完依赖后将target置为空
   		Dep.target = null;
   	}
   	/**
   	 * 更新视图
   	 */
   	update() {
   		this.getVal();
   		this.node.nodeValue = this.value;
   	}
   	/**
   	 * 触发访问器中的get，从而向Dep中添加watcher，达到依赖收集的目的。
   	 */
   	getVal() {
   		this.value = this.vm[this.name];
   	}
   }
   
   class Dep {
   	constructor() {
   		this.subs = [];
   	}
   
   	depend(sub) {
   		this.subs.push(sub);
   	}
   
   	notify() {
   		this.subs.forEach((sub) => {
   			sub.update();
   		})
   	}
   }
   Dep.target = null;
   ```

   这是修改之后的访问器属性配置：

   ```js
   	defineReactive(prop, value) {
   		let dep = new Dep();
   		Object.defineProperty(this, prop, {
   			get: function() {
   				if(Dep.target) {
   					dep.depend(Dep.target);
   				}
   				return value;
   			},
   			set: function(newValue) {
   				if(value === newValue) return;
   				value = newValue;
   				dep.notify();
   			}
   		})
   	}
   ```

   可以看到，在配置data中某个属性的访问器时就已经实例化了Dep，不要忘了在compile的时候实例化Watcher。

   ```js
   	compile(node) {
   		let reg = /\{\{(.*)\}\}/;
   		if(node.nodeType === 1) {
   			let attrs = node.attributes;
   			for(let i = 0; i < attrs.length; i++) {
   				if(attrs[i].nodeName == 'v-model') {
   					node.addEventListener('input', (e) => {
   						this[attrs[i].nodeValue] = e.target.value;
   					})
   					node.value = this[attrs[i].nodeValue];
   				}
   			}
   		} else if(node.nodeType === 3) {
   			if(reg.test(node.nodeValue)) {
   				let dataKey = RegExp.$1;
   				dataKey = dataKey.trim();
   				node.nodeValue = this[dataKey];
   				new Watcher(node, dataKey, this);
   			}
   		}
   	}
   ```



## 总结

Vue.js中很便捷的一个特性用了不过一百行左右就实现了，这个简单的实现还无法处理稍微复杂一些的情况，但是其中的观察者模式的思想和对访问器属性的使用也是让我受益良多。

刚刚开始学习Vue.js的源码，如果文中有错误还请多多指教。

联系方式：ziixuanwu@gmail.com

源码：[bind.html](https://github.com/ZacheryWu/vue-learn-demo/blob/master/bind.html)

参考资料：

- [vue-come-true](https://github.com/coderzzp/vue-come-true/tree/master/vue-come-true-First)
- [Vue 源码解析：深入响应式原理](https://github.com/DDFE/DDFE-blog/issues/7)
- [响应式系统的基本原理](https://gaodaqian.com/booklet-vue-core/0_Vue.js%E8%BF%90%E8%A1%8C%E6%9C%BA%E5%88%B6%E5%85%A8%E5%B1%80%E6%A6%82%E8%A7%88.html#%E5%85%A8%E5%B1%80%E6%A6%82%E8%A7%88)





