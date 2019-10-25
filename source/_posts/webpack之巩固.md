---
title: webpack之巩固
date: 2019-10-23 14:54:32
tags:
  - webpack
---

## 概念

> 本质上，webpack 是一个现代 JavaScript 应用程序的静态模块打包工具。当 webpack 处理应用程序时，它会在内部构建一个**依赖图(dependency graph)**，此依赖图会映射项目所需的每个模块，并生成一个或多个 bundle。

上面一段话摘自官方文档，由此引申出下面几个问题

1. 什么是依赖图？依赖图是怎么映射项目所需的模块？
2. bundle 是什么？

<!-- more -->

### 依赖图

webpack 开箱即用，可以无需使用任何配置文件。然而，webpack 会假定项目的入口起点为工程目录 `src/index`，然后会在 `dist/main.js` 输出结果，并且在生产环境开启压缩和优化。

也就是说，在没有任何配置文件和命令行传参的情况下，会有一个默认的入口起点 `src/index` (如果自己配置了入口就会使用配置好的)，直接执行

```cmd
webpack
```

webpack 会将 `mode` 的默认值设置为 `production`并开始打包，从 入口起点 开始，webpack 递归地构建一个依赖图，这个依赖图包含着应用程序所需的每个模块，然后将所有这些模块打包为少量的 bundle - 通常只有一个 - 可由浏览器加载。

所以依赖图的生成是首先取决于入口文件，当然，在入口文件中如果也引入了其它文件，那么其它文件也会变成依赖图一部分。

### bundle

bundle 就是 webpack 生成的文件，bundle 里包含多个 chunk - 代码块，可能多个 bundle 会存在相同的代码块，所以需要用代码分离来共享相同代码块部分

## 入口 entry

入口的写法，

```javascript
// webpack.config.js
module.exports = {
  // normal
  entry: './path/to/my/entry/file.js'
  // 对象语法
  // 使用 optimization.splitChunks 为页面间共享的应用程序代码创建 bundle。
  // 由于入口起点增多，多页应用能够复用入口起点之间的大量代码/模块
  // 如果输出为library 可以使用 array 语法暴露所有模块
  // 参考 https://github.com/webpack/webpack/tree/master/examples/multi-part-library
  entry: {
    main: './path/to/my/entry/file.js'
  }
  // 数组
  // 这样传会合并多个文件为一个bundle文件
  // 如果输出为library时只会暴露最后一个模块
  entry: ['path1', 'path2']
  // 动态入口 可以接收远程服务器的数据来决定入口，感觉没什么用
  entry: () => './demo'
  // 或
  entry: () => new Promise((resolve) => resolve(['./demo', './demo2']))
};


```

## 输出 output

输出主要是告诉 webpack 打包后的 bundle 放在哪里，以及如何命名这些文件

```javascript
const path = require('path')

module.exports = {
  // ...
  // normal
  output: {
    path: path.resolve(__dirname, 'dist'), // 打包后的目录 URL以HTML页面为基准
    filename: 'my-first-webpack.bundle.js', // 文件命名
    publicPath: 'assets/', // 相对于 html 页面
    publicPath: '/assets/' // 相对于服务器根目录
  },
  // 多个入口起点 使用占位符
  output: {
    filename: '[name].js'
    // 输出文件 ./dist/name1.js  ./dist/name2.js
  },
  // cdn hash
  output: {
    path: '/home/proj/cdn/assets/[hash]', // 此处打包后会生成到硬盘根路径
    publicPath: 'http://cdn.example.com/assets/[hash]/' // cdn路径
  }
}
```

### 在运行时设置 publicPath

所谓运行时，即在打包完成后运行应用程序的时候。一般在 output 中配置的 publicPath 是固定的，但是，我们可能需要在运行的时候动态加载 publicPath,webpack 暴露了一个名为 **webpack_public_path** 的全局变量，通过改变这个变量的值达到我们的目的。

1. 创建一个文件`public_path.js`

```javascript
__webpack_public_path__ = 'http://some.cdn.com/some'
```

2. 在入口文件中引入

```javascript
// entry.js
import 'public_path.js'
import './app.js'
```

> 如果在 entry 文件中使用 ES2015 module import，则会在 import 之后进行 **webpack_public_path** 赋值。在这种情况下，你必须将 public path 赋值移至一个专用模块中，然后将它的 import 语句放置到 entry.js 最上面

### chunkFilename

定义非入口 chunk 文件的名称。这个在动态导入时可以设置分出来的文件名

如下，在文件中使用 import()时，webpack 会在打包时将 a.js 分离出去，成为一个新的文件，这个文件在没有设置 chunkFilename 时会自动使用`[模块id].js`命名(如果注释名存在则使用注释名), 当然，使用注释命名也可以

```javascript
// entry.js
import('./a.js') // 输出文件 0.js
import(/* webpackChunkName: "chunk1" */'./a.js') // 输出文件 chunk1.js

// 设置chunkFilename后
output: {
  chunkFilename: '[name].[chunkhash].js' // name 一般是模块id 如果有注释名则使用注释名
},

// 输出
// 0.[chunkhash].js
// chunk1.[chunkhash].js
```

### crossOriginLoading | jsonpScriptType | chunkLoadTimeout

crossOriginLoading，只用于 `target` 是 `web`，使用了通过 script 标签的 JSONP 来按需加载 chunk。通过加载资源的 origin 信息来判断是否跨域，比如在 cdn 加载 chunk 的时候肯定是跨域的，那么此设置就会生效

jsonpScriptType 设置 jsonp 中 script 的 type 属性

chunkLoadTimeout 设置 script 中超时时间，默认 120s

```javascript
if (script.src.indexOf(window.location.origin + '/') !== 0) {
  script.crossOrigin = 'anonymous'
}
```

### filename 中的 chunkhash contenthash

chunkhash 和 contenthash 的区别在于，都是 chunk 内容，不过 contenthash 是通过`ExtractTextWebpackPlugin`提取出来的 css hash，用于 css 文件的命名

### libraryTarget

配置如何暴露 library。

1. **var**. （默认值）当 library 加载完成，入口起点的返回值将分配给 library 变量，会覆盖掉已经定义过的全局变量（谨慎使用）

```javascript
output.library = 'someLibName'
// 打包后，加载完库后会把库对象分配给全局变量 someLibName
var someLibName = module.exports // 输出结果，如果在之前存在全局变量someLibName会覆盖
```

2. **assign**. 比 `'var'`少了个 var，可以说没区别

```javascript
someLibName = module.exports // 输出结果
```

3. **this**.

- output.library 没有赋值，webpack 将把 library 对象上所有的属性挂载到浏览器的 this 上，也就是 window

```javascript
(function(e, a) { for(var i in a) e[i] = a[i]; }(this, module.exports)
// 遍历exports对象并挂载到this
```

- `output.library = 'someLibName'`则会将对象挂载到`this['someLibName']`

```javascript
this['someLibName'] = module.exports
```

4. **window** 同上

```javascript
window['someLibName'] = module.exports
```

5. **global** 分配给 global 对象

```javascript
global['someLibName'] = module.exports
```

6. **commonjs** 分配给 exports 对象。这个名称也意味着，模块用于 CommonJS 环境，在浏览器下不可用

```javascript
exports['someLibName'] = module.exports

require('someLibName').doSomething()
```

7. **commonjs2** 模块定义系统.用于`CommonJS`系统，入口起点的返回值将分配给 `module.exports` 对象。

与`commonjs`的区别是不用指定 output.library

> 模块定义系统会使 `bundle` 带有更多的头部处理，以便兼容各种模块系统

```javascript
module.exports = _entry_return_

require('MyLibrary').doSomething()
```
