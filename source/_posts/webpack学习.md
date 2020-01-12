---
title: webpack学习
date: 2019-12-22 20:41:00
tags:
---

看了[这篇](https://juejin.im/post/5de87444518825124c50cd36#heading-6)文章，自己敲了两遍，现在试着输出一下。
## 基本配置

### 初始化项目

首先创建一个空的项目，使用`npm init` 初始化一下，这是个webpack的学习项目，因此首先要做的就是安装webpack了。

```
npm i -D webpack webpack-cli
```

初步建立一下文件的目录

```
├─build //webpack配置文件
├─public //公共的文件
└─src //源码
```

在src下添加一个main.js，随便写点代码：

```js
console.log('我能吞下玻璃而不伤身体')
```

试着打包一下这个项目，在package.json中添加build命令：

```j
"build": "webpack src/main.js"
```

这时就可以在执行`npm run build `来打包src中的main.js了。

执行完之后可以看到项目目录下多了个dist文件夹，文件夹下面有个main.js文件。

这样的打包还是功能太少了，无法满足我们日常开发的需求，为了更好的配置webpack，接下来创建webpack的配置文件。

### 编写自己的配置文件

在build文件夹中建立`webpack.conf.js` 配置文件

```js
const Path = require('path')

module.exports = {
    mode: 'development', //配置为开发模式
    entry: Path.resolve(__dirname, '../src/main'), //项目的入口文件
    output: {
        filename: 'output.js', //项目输出文件名
        path: Path.resolve(__dirname, '../dist')//项目输出文件路径
    }
}
```

然后修改package.json 将build命令的配置指向webpack.conf.js

```
"build": "webpack --config build/webpack.conf.js"
```

执行npm run build 之后可以发现dist文件下多了一个output.js

但是之前的main.js并没有消失，为了不用我们每次手动清理，添加插件`clean-webpack-plugin`来清理dist文件夹。配置文件变成了：

```js
const Path = require('path')
const {CleanWebpackPlugin} = require('clean-webpack-plugin')
module.exports = {
    mode: 'development',
    entry: Path.resolve(__dirname, '../src/main'),
    output: {
        filename: 'output.js',
        path: Path.resolve(__dirname, '../dist')
    },
    plugins: [
        new CleanWebpackPlugin(),
    ]
}
```

插件的作用：在webpack的运行周期中会广播很多事件，Plugin可以监听这些事件，在合适的时机通过Webpack暴露出的API改变输出的结果。

### 引入模板

前端开发中通常是有一个html文件作为项目的入口的，我们虽然把js打包好了，但是每次在html中手动引入js是及其不方便的，尤其是通常会把输出的文件带上该文件的哈希值来避免浏览器缓存而不加载文件，有个插件可以帮助我们实现这个功能，`html-webpack-plugin` 。

下载完之后添加插件到配置中：

```js
const Path = require('path')
const HtmlWebpackPlugin = require('html-webpack-plugin')
const {CleanWebpackPlugin} = require('clean-webpack-plugin')
module.exports = {
    mode: 'development',
    entry: Path.resolve(__dirname, '../src/main'),
    output: {
        filename: '[name].[hash].js',
        path: Path.resolve(__dirname, '../dist')
    },
    plugins: [
        new CleanWebpackPlugin(),
        new HtmlWebpackPlugin({
            template: Path.resolve(__dirname, '../public/index.html')
        })
    ]
}
```

执行 npm run build 命令，可以看到dist文件夹下面多了一个index.html 文件，并且在这个文件中引入了打包之后的js文件。

### 多入口文件

大型项目中，一个模板文件通常并不能满足我们的需求，这时可以通过添加多个`html-webpack-plugin`的实例来实现：

```js
const Path = require('path')
const HtmlWebpackPlugin = require('html-webpack-plugin')
const {CleanWebpackPlugin} = require('clean-webpack-plugin')
module.exports = {
    mode: 'development',
    entry: {
        main: Path.resolve(__dirname, '../src/main'), //入口文件改为多个
        another: Path.resolve(__dirname, '../src/another')//key(main和another是chunk的名字)
    },
    output: {
        filename: '[name].[hash].js',
        path: Path.resolve(__dirname, '../dist')
    },
    plugins: [
        new CleanWebpackPlugin(),
        new HtmlWebpackPlugin({
            template: Path.resolve(__dirname, '../public/index.html'),
            filename: 'index.html',
            chunks: ['main']
        }),
        new HtmlWebpackPlugin({ //另一个实例
            template: Path.resolve(__dirname, '../public/another.html'),
            filename: 'another.html',
            chunks: ['another']
        })
    ]
}
```

### 引入css

css的配置需要添加loader，loader用来对不同的类型的文件进行处理。处理css需要用到`css-loader` `style-loader` 。添加loader不需要在配置文件中require，只需要在module中添加规则和对应的loader即可。

```js
module.exports = {
    //...其他配置省略
    module: {
        rules: [
            {
                test: /\.css$/,
                use: ['style-loader', 'css-loader'] //loader是栈的结构，从后往前解析
            }
        ]
    },
}
```

#### 引入less

less的引入只需要在css的配置上添加另一个loader即可：

```js
module.exports = {
    //...其他配置省略
    module: {
        rules: [
            {
                test: /\.css$/,
                use: ['style-loader', 'css-loader'] //loader是栈的结构，从后往前解析
            },
            {
                test: /\.less$/,
                use: ['style-loader', 'css-loader', 'less-loader']
            }
        ]
    },
}
```

不过需要注意的是，只有在js中引入的css或less文件才会被打包到最终的目标文件中，且样式是以js代码的形式存在的。

### 抽取样式代码

使用`mini-css-extract-plugin` 把css从js中提取出来，打包到单独的css文件中，方便查看(虽然一般不会去看)，配置如下:

```js
const Path = require('path')
const HtmlWebpackPlugin = require('html-webpack-plugin')
const MiniCssExtractPlugin = require('mini-css-extract-plugin')
const {CleanWebpackPlugin} = require('clean-webpack-plugin')

module.exports = {
    //...其他配置已省略
    output: {
        filename: 'static/js/[name].[hash].js',
        path: Path.resolve(__dirname, '../dist')
    },
    module: {
        rules: [
            {
                test: /\.css$/,
                use: [MiniCssExtractPlugin.loader, 'css-loader']
            },
            {
                test: /\.less$/,
                use: [MiniCssExtractPlugin.loader, 'css-loader', 'less-loader']
            }
        ]
    },
    plugins: [
        new MiniCssExtractPlugin({
            filename: 'static/css/[name].[hash].css',
            chunkFilename: '[id].css'
        })
    ]
}
```

这时候打包之后的文件已经比较多了，可以在filename中添加路径把不同类型的文件分类打包。

### 添加babel

安装babel：

```
npm i -D babel-loader @babel/preset-env @babel/core
```

@babel/preset-env 用于预设环境，使得使用者可以使用最新的JavaScript而不用在不同的环境之间切换babel版本。

下面是babel的配置：

```
module: {
        rules: [
            {
                test: /\.js$/,
                use: {
                    loader: 'babel-loader',
                    options: {
                        presets: ['@babel/preset-env']
                    }
                },
                exclude: /node_modules/
            },
       ]
}
```

有些奇怪的是，我把这个配置以及babel处理相关的包卸载掉以后，webpack依然能够解析ES6的语法。？？？还没搞懂什么情况。

### 对图片、字体等文件的处理

主要是通过url-loader和file-loader这两个loader来进行加载。

url-loader的作用是把资源转化为base64格式，可以减少网络请求。但是对于比较大的资源，转化为base64会导致性能降低。因此url-loader有个配置limit，用来设置最大的文件大小，如果资源大小超过限制，那么将采用fallback中配置的loader来进行加载。(默认为file-loader)

file-loader的作用是解析import或require中的资源，并把这些资源移动到dist文件夹中。

这里为了简单说明，只设置了png格式的文件配置：（其他类型相同）

```js
module: {
        rules: [
            {
                test: /\.png$/,
                use: {
                    loader: 'url-loader',
                    options: {
                        limit: 10000,
                        fallback: {
                            loader: 'file-loader',
                            options: {}
                        }
                    },
                }
            }
       ]
}
```



以上就是webpack最基本的配置，主要是entry、output、module(loader的配置)、plugins属性的配置。虽然已经敲了三遍了...但真到用的时候还是需要去找这些loader和plugin的名字，不过也算是对webpack中的基本配置有了一定的了解。

## Vue开发环境搭建

通过添加对应的loader、plugin等完成Vue项目的开发环境搭建。

### .vue文件的解析

安装`vue-loader`，添加到module中，并且添加Vueloader的插件到plugin中，如果感觉打包过慢，可以使用cache-loader等对vue的打包加速。

```js
const VueLoaderPlugin = require('vue-loader/lib/plugin')
module.exports = {
    module: {
        rules: [
            {
                test: /\.vue$/,
                loader: ['vue-loader']
            }
        ]
    },
    plugins: [
        new VueLoaderPlugin()
    ]
}
```

在平时的开发中还经常使用一些路径的简写比如用@来表示源代码的根目录，这个可以根据resolve下的alias属性来配置。

### 文件路径别名配置

resolve配置是用来设置模块如何进行解析的，直观点说就是js文件中的import命令解析时所采用的配置。下面使用一下alias和extensions属性：

```js
const Path = require('path')
module.exportx = {
    resolve: {
        alias: {
            ' @': Path.resolve(__dirname, '../src'),
            'vue$': 'vue/dist/vue.runtime.esm.js',
        },
        extensions: ['.js', '.json', '.vue']
    },
   
}
```

配置完上面这些属性之后，`import '@/components/Button'` 就和 `import '../../src/components/Button.vue'`效果相同。

在添加完这些基本配置之后就可以对Vue文件进行打包了，不过还是无法像vue-cli一样启动一个服务并且热更新，其实vue-cli也是使用webpack的server服务，只需要添加devServer配置就可以实现了。

### devServer的配置

这个server的本质就是一个文件服务器，所以在配置时服务所在的根目录，以及服务所占用的端口，如果需要热更新的话，还要配置一下热更新，然后在添加一下大名鼎鼎的HMR插件就可以了：

```js
const Webpack = require('webpack')
module.exports = {
    devServer: {
        port: 3000,
        contentBase: '../dist',
        hot: true
    },
    plugins: [
        new Webpack.HotModuleReplacementPlugin()
    ]
}
```

配置完之后，再修改一下package.json就可以启动自己的服务了。在package.json中添加下面这一行命令：

```js
"dev": "webpack-dev-server --config build/webpack.conf.js --open"
```

然后就可以通过npm run dev 来启动自己的服务。

配置没错的话，就可以在浏览器中的 localhost:3000这个地址看到你的页面了。

但是诡异的事情发生了，dist文件夹下面打包后的文件全部失踪了，这是为什么呢？

原来是之前配置的CleanWebpackPlugin搞的鬼，在我们启动webpack-dev-server的时候也清空了dist文件夹，但是却没有再生成目标文件，就导致dist文件夹是空的。但是在执行npm run build 命令时又确实需要情况dist文件夹，为了解决这个问题，把执行打包命令和启动服务命令的配置分开写。

### 拆分dev和prod的配置

虽然dev和prod的配置有些不同，但是还是有大部分是相同的，因此还是使用原本的webpack.conf.js作为基本配置，然后webpack.dev.js和webpack.prod.js作为开发和打包配置。为了把webapck.conf.js和webapck.dev.js中的配置结合起来，需要引入webpack-merge 这个工具来进行合并。

然后把devServer相关的配置迁移到webpack.dev.js中，把CleanWebpackPlugin迁移到webpack.prod.js中。得到的配置文件为：

```js
//webpack.dev.js
const WebpackMerge = require('webpack-merge')
const WebpackBase = require('./webpack.conf')
const Webpack = require('webpack')

module.exports = WebpackMerge(WebpackBase, {
    mode: 'development',
    devServer: {
        port: 3000,
        hot: true,
        contentBase: '../dist'
    },
    plugins: [
        new Webpack.HotModuleReplacementPlugin()
    ]
})
```

```js
//webpack.prod.js
const WebpackMerge = require('webpack-merge')
const WebpackBase = require('./webpack.conf')
const {CleanWebpackPlugin} = require('clean-webpack-plugin')

module.exports = WebpackMerge(WebpackBase, {
    mode: 'production',
    plugins: [
        new CleanWebpackPlugin(),
    ]
})
```

最后别忘了去package.json中修改命令的配置：

```
"build": "webpack --config build/webpack.prod.js",
"dev": "webpack-dev-server --config build/webpack.dev.js --open"
```

这下再启动server ，dist下的文件已经不会消失了。

### 分析打包体积

使用`webpack-bundle-analyzer`进行打包体积的分析。

`webpack-bundle-analyzer` 可以很直观地展示出打包之后不同模块所占用的体积，使用也很方便，只需要添加插件就可以在打包完成之后自动打开一个web服务，来展示体积信息。

```js
const {CleanWebpackPlugin} = require('clean-webpack-plugin')
const BundleAnalyzerPlugin  = require('webpack-bundle-analyzer').BundleAnalyzerPlugin
let WebpackProdConf = WebpackMerge(WebpackBase, {
    mode: 'development',
    plugins: [
        new CleanWebpackPlugin(),
        new BundleAnalyzerPlugin()
    ]
})
```

效果如下：



但是还有个小问题，就是这样打包之后每次打包都会打开这个分析工具，这个问题解决起来也很简单，我们可以通过命令传入的参数来进行判断是否需要体积分析：

```js
const WebpackMerge = require('webpack-merge')
const WebpackBase = require('./webpack.conf')
const {CleanWebpackPlugin} = require('clean-webpack-plugin')
let WebpackProdConf = WebpackMerge(WebpackBase, {
    mode: 'development',
    plugins: [
        new CleanWebpackPlugin(),
    ]
})
if(process.env.npm_config_report) {
    const BundleAnalyzerPlugin  = require('webpack-bundle-analyzer').BundleAnalyzerPlugin
    WebpackProdConf.plugins.push(new BundleAnalyzerPlugin())
}
module.exports = WebpackProdConf
```

配置好之后，就可以通过`npm run build --report `来打开体积分析了。

在npm命令中，会把`--`开头的参数传递到npm本身，而不是传给对应的指令，并且这个参数可以通过`process.env.npm_conifg_`获取到。

## 总结
现在再看到webpack的配置不是一头包了，也能按照不同的模块去分析分析，所欠缺的是熟练度以及对于不同loader和plugin的了解，以后多看多写，一定会掌握更多的。至于loader和plugin的编写，看着教程也写了一下，对loader和plugin的理解倒是加深了一些，如果有需要去写loader或者plugin，看下文档相信也可以很快上手的。


## 参考资料
- [weboack官方文档](https://webpack.docschina.org/concepts/)
- [2020年了,再不会webpack敲得代码就不香了](https://juejin.im/post/5de87444518825124c50cd36#heading-6)
- [入门 Webpack，看这篇就够了](https://segmentfault.com/a/1190000006178770)
- [深入Webpack-编写Loader](https://segmentfault.com/a/1190000012718374)
