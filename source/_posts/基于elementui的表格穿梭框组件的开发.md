---
title: 基于elementui的表格穿梭框组件的开发
date: 2019-08-21 22:21:43
tags:
---
在开发过程中需要用到一个表格的穿梭框，但发现elementui中的穿梭框不支持自定义，本着不重复造轮子的原则去npm上找了一圈，并没有找到，那只能自己撸一个了。最终的效果是这样的：
![table_transfer_result.gif](https://i.loli.net/2019/08/27/UZl28uzeiCgtsIP.gif)
其实这个组件很简单，两个可以复选的表格，以及两个控制移动的按钮就完成了，不过后续也在这个的基础上添加了一些功能，如禁用某一行的勾选，对表格中数据显示的自定义，等等。

因为想要尝试一下将组件发布到npm上，所以和平时的Vue组件的流程并不完全相同，而是多了一些配置上的操作。

整体流程分为四步：

1. 搭建组件开发环境
2. 开发组件
3. 编写demo
4. 发布到npm

## 搭建组件开发环境

在整个开发过程中，环境的搭建对于没有发布过组件的我来说才是最复杂的。

前面也提到了想要发布到npm上的话并不能仅仅开发一个单页面组件，而是要对这个组件进行一定的封装，并且经过webpack的打包，最好还能够支持多种引入的方式。

这是我的文件目录：

![1566779888233](G:\hexo\blog\source\images\1566779888233.png)

代码部分主要由两部分组成,examples和packages。顾名思义，examples中放的是示例文件，packages中放的则是我们这次开发的组件。之所以要把elTableTransfer 放到packages里面，是因为在组件的开发中可能会有这一套组件不止一个的情况，这种情况下，就可以把整套组件从packages中打包、导出。

使用vue-cli3创建的文件结构是这样的：

![1566780195124.png](https://i.loli.net/2019/08/27/J5Bthvrx8q4HlXA.png)

和想要的结构还不一样。在使用vue-cli3创建项目完成后，把`src` 改为`examples` ，然后添加另一个文件夹`packages` 。改了文件结构之后，整个项目的入口变了，要通过babel转换的文件路径也变了。

因此要修改webpack的打包方式，vuecli3中把webpack中的配置移到了`vue.config.js`中。

首先添加项目的入口：

``` js
module.exports = {
	pages: {
		index: {
			entry: 'examples/main.js',
			template: 'public/index.html',
			filename: 'index.html'
		}
	},
}
```

然后把新添加的`packages` 添加到babel中。在vuecli3中，这些配置并不能像配置项目入口那样，直接配置，而是通过[链式操作](https://github.com/neutrinojs/webpack-chain)来实现:

``` js
module.exports = {
	pages: {
		index: {
			entry: 'examples/main.js',
			template: 'public/index.html',
			filename: 'index.html'
		}
	},
	chainWebpack: (config) => {
		config.module
			.rule('js')
			.include
				.add('/packages')
				.end()
			.use('babel')
				.loader('babel-loader')
				.tap(option => option)
	}
}
```

## 组件开发

这个穿梭框组件分为三个部分，表格 按钮 表格。因为是基于elementui的，所以也不用费力去自己写表格，调样式，还可以用elementui的布局。所以将整个组件作为一个row，其中有三个col。

#### 左半部分表格

按着顺序一个一个来写，首先是第一个col，左半部分表格。为了让内容主题更加突出，我把table放到了el-card中，用el-card的头部来表明这个表格的主题。然后是表格的部分，穿梭框其实并不需要用到太多复杂的功能，只需要加一个多选列就ok了。但是考虑到可能会有自定义表格显示的需求，我模仿elementui的表格，使用插槽的方式从父组件中接收摸板，然后放到elementui的表格插槽中，对，我只是个搬运工😜。

#### 中间部分按钮

中间按钮也很简单，放一左一右两个箭头，然后稍稍调整下样式就可以了。样式的点击事件也并不复杂。

- 向👉箭头的点击事件，需要先把被勾选的数据加到右边表格的数据中，然后把被勾选的数据从左边表格的数据中删除就可以了。
- 向👈箭头的点击事件，和向右的类似，不再赘述。

由于从父组件传递过来的数据是数组，引用类型，所以直接修改的话就可以影响到父组件，并不需要担心向父组件传值的问题。

#### 右半边部分表格

和左边表格类似，且由于是穿梭框，两个表格的列信息肯定是一样的，所以两个表格公用一个列的配置。除了数据源和表格的主题和左边的完全相同。

花费不多的时间组件就开发完了。源码在[这里](https://live.bilibili.com/528)



在组件开发好之后，还需要按照Vue的标准，给这个组件封装上install方法：

```js
import elTableTransfer from './src/elTableTransfer.vue'

elTableTransfer.install = function(Vue) {
	Vue.component(elTableTransfer.name, elTableTransfer);
}

export default elTableTransfer;
```

不知道大家是否还记得，在搭建环境的时候，考虑到组件可能不止一个，还在组件外面包了一层，那么这一层就是要对内部的所有组件做一下封装，做成一个统一的install方法，方便使用者注册组件：

```js
import elTableTransfer from './elTableTransfer'

const components = [ elTableTransfer ];

function install(Vue) {
	components.map(component => {
		Vue.component(component.name, component);
	})
}

if(typeof window !== undefined && window.Vue) {
	install(window.Vue);
}

export default {
	elTableTransfer,
	install
}
```

到这里，组件的开发就基本完成，但是在上传之前总要自己先验证一下效果，所以开发个demo试用一下新开发组件。（我在开发中其实是是先写好demo，然后开发组件时就可以一边开发一边看到效果了，这里的顺序是为了方便描述）

## 编写demo

只需要在demo中用到上面编写的组件就可以了，需要注意的是，demo中记得引入elementui，因为组件是基于elementui的。



## 发布

vue-cli-service build可以通过--target选项指定不同的[构建目标](https://cli.vuejs.org/zh/guide/build-targets.html)。

默认的模式是应用模式，在这个模式中

- `index.html` 会带有注入的资源和 resource hint
- 第三方库会被分到一个独立包以便更好的缓存
- 小于 4kb 的静态资源会被内联在 JavaScript 中
- `public` 中的静态资源会被复制到输出目录中

我们发布需要修改为库模式，在库模式中，Vue是外置的，这意味着包内不会有Vue。可以通过下面的命令将一个单独的入口构建为一个库：

`vue-cli-service build --target lib --name myLib [entry]`

因此在package.json中添加命令：

`"lib": "vue-cli-service build --target lib --name eltabletransfer --dest lib packages/index.js"`

打包结果：

![1566868076154](G:\hexo\blog\source\images\1566868076154.png)

在发布之前还要注意一下package.json中的一些信息，比如name、version等等，还有private必须是false，否则是无法发布的。除此之外还要添加一个`.npmignore`文件，用来忽略一些不想发布上去的文件和文件夹。

接下来就登陆自己的npm账号

![1566913492883](G:\hexo\blog\source\images\1566913492883.png)

然后执行`npm publish` 命令，等上几秒钟，就可以在npm上看到自己发布的组件了!

![1566913595334](G:\hexo\blog\source\images\1566913595334.png)



## 项目地址

[npm地址](https://www.npmjs.com/package/el-table-transfer)

[github地址]()

## 参考文章

[详解：Vue cli3 库模式搭建组件库并发布到 npm](https://juejin.im/post/5bbab9de5188255c8c0cb0e3)

[Vue cli3 构建目标]([https://cli.vuejs.org/zh/guide/build-targets.html#%E5%BA%94%E7%94%A8](https://cli.vuejs.org/zh/guide/build-targets.html#应用))

[Vue cli3 webpack配置](https://cli.vuejs.org/zh/guide/webpack.html)