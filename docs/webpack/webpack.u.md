# webpack
## 打包成了什么
如果我们用 webpack 默认(不加任何配置)配置打包一个简单项目, 会发现
产出的内容出现在 ./dist 下, 打开看看, 就是一个 IIFE(立即执行函数)
```js
(function (modules) {
  // ...
  // 闭包变量, 用于缓存已经加载的模块
  const installedModules = []
  // webpack 模块加载函数
  const __webpack_require__ = function () {
    // ...
  }
  // ...
})({
  "./src/hello.js": (function (module, exports) {
    // hello.js 代码
  }),
  "./src/index.js": (function (module, exports) {
    // index.js 代码
  })
})
```
这个 IIFE 被人称为 webpackBootstrap
它接收了一个 modules 列表, modules 的 key 为模块路径, value 是经过处理过后的我们的代码

## ES Module
Webpack 默认是可以处理基于 CommonJs 的模块化写法, 如果我们用的 ES module, 会发生什么呢?

会多出来一个 
```js
__webpack_require__.r(__webpack_exports__)
```
它是来给模块的 exports 对象上加上使用 ESM 模块规范的标记的 (exports.__esModule = true)
有了标记, webpack 还设计了一些方法来为带有 esModule 标记的 exports 改造成符合 ESM 规范的模样

## 按需加载
在 webpack.config.js 中加入相关 babel 插件
```js
module.exports = {
  module: {
    rules: [
      test: /\.js$/,
      exclude: /node_modules/,
      loader: "babel-loader",
      options: {
        "plugins": [
          "dynamic-import-webpack"
        ]
      }
    ]
  }
}
```
同时, 在 index 文件中使用动态 import 
```js
// hello.js
export default sayHello = name => 'hello' + name

// index.js
import('./hello.js').then(sayHello => {
  console.log(sayHello('maos'))
})
```
然后用 webpack 打包, 发现输出了两个文件, 分别是主文件 main.js 和一个异步加载的 0.js
0.js 里除了 hello.js 的内容以外, 还有一些配合 main.js 进行异步加载的代码, 大概就是:
  webpack 发现这个模块是异步的, 然后按配置的路径, 使用 Promise 异步载入 script
  0.js 的代码会通过 JSONP 的方式加载, 然后执行, 然后和普通模块别无二致了

**CommonsChunkPlugin**
这个插件用来分割第三方依赖库的代码, 将业务逻辑和稳定的库脚本分离, 以优化代码体积, 合理利用缓存
实际上, 它的思路与上述按需加载的方式是一样的

## webpack 工作流程
首先, 读取 webpack.config.js, 或者从 shell 中获得必要的参数
接着, 根据配置, 实例化 compiler 对象, 实例化所有的 plugins, 在 webpack 事件流上挂载钩子
同时, 从配置的入口文件开始, 收集依赖, 利用配置的 loaders 对依赖进行处理
  将处理好的内容使用 acorn 进行编译, 生成 AST (抽象语法树)
  利用 AST 把不同的模块加载语法替换为 __webpack_require__
  把新 AST 合成为产出结果输出到配置的目录文件中

## AST
AST 就是将代码转换为树结构的 object 集合

## compiler 和 compilation
全局只存在一个 compiler 实例, 它就像是 webpack 的神经中枢
我们的 plugins 通过这个对象就可以访问 webpack 的内部环境, 包括全部配置和事件钩子

compilation 对象包含了当前的模块资源, 编译生成的资源, 变化的文件等等信息
在 webpack 以 development 模式运行时, 每当检测到文件变化时, 一个崭新的 compilation 对象就会被创建
所有构建过程中产生的构建数据都会被存储到换个对象上, 而这个对象上也有事件钩子

compiler 和 compilation 都组合 (也可以说继承) 了 tapable 库, 因而都提供了很多事件回调供 plugins 做扩展

compiler 和 compilation 感觉像是一个套娃, 他们有相似的能力, 好像在做同一件事情, 但是又必须分开用
我认为本质上二者都是 webpack 的事件流
  compiler 代表 webpack 中每次构建不变的东西 (webpack 本身的执行流程)
  compilation 负责可变的东西 (具体构建内容时的事件流)
每一次的构建流程 compiler 都是同一个, 而 compilation 则会重新被创建, 来给 compiler 打工
我们的 plugin 需要先响应 compiler 的事件, 然后这个事件响应中再去响应 compilation 的事件

compiler 提供的常用钩子, [官网](https://www.webpackjs.com/api/compiler-hooks/):
```js
compiler.hooks.entryOption.tap(...) // SyncBailHook 钩子, 在 webpack 读取entry 配置后触发
compiler.hooks.emit.tap(...) // AsyncSeriesHook 钩子, 在生成的资源输出之前触发
```

compilation 提供的常用钩子, [官网](https://www.webpackjs.com/api/compilation-hooks/):
```js

```

此外, compiler 和 compilation 上面也可以加上自定义的钩子

## tapable
[官网](https://github.com/webpack/tapable)
tapable 是一个自定义事件库, 其能力类似于 NodeJs 的 EventEmitter 类
tapable 库比 EventEmitter 更加复杂且神通广大, 它有 同步/异步/Bail/Waterfall/Loop 多种类型钩子
tapable 的具体实现, [还有待进一步研究](../../studys/source_code/tapable.md)

## plugins
[官网](https://www.webpackjs.com/api/plugins)
webpack 事件流发挥作用的地方, 这一设计的核心在与: 
plugins 并不直接操作文件, 而是基于事件机制, 监听 webpack 打包过程中的某些事件
见缝插针, 影响打包结果

那么如何设计一个 webpack 插件呢? 
  首先要想清楚这个插件解决什么问题
  然后根据问题找出相应的事件钩子
  在相应事件中搞事情, 解决问题

插件还必须能够与 webpack 相配合, webpack 要求它是一个带有 apply 方法的类, 一个基本写法是:
```js
class CustomPlugin {
  constructor(options) {
    this.options = options
  }
  apply(compiler) {
    // 注册 compiler 某个事件回调
    compiler.hooks.someHook.tap(this.constructor.name, (compilation, callback) => {
      // magic ...
      // 如果需要的话再添加 compilation 钩子
      compilation.hooks.someOtherHook.tap('SomePlugin', () => {
        // magic ...
      })
    })
  }
}

module.exports = CustomPlugin
```
plugin 也支持一些钩子回调执行异步操作, 通过把 tap 方法换成 tapAsync 或 tapPromise 来实现

## loaders
在 webpack 中, loader 是真正发生魔法的地方:
  Babel 将 ES Next 代码编译成 ES5
  less-loader 将 less 编译成 css
  ...

loaders 的设计还秉持单一职责, 只完成最小单元的文件转换任务
  比如, less 文件会先通过 less-loader 输出 css, 之后交给 css-loader 处理
  甚至还需要将 css-loader 输出的内容交给 style-loader 处理创建 style node
  像 compose 一样串联起来执行, 每一个 loader 只关心自己的输入和输出

所以 loader 的本质就是一个函数:
```js
// webpack 提供的工具集
const loaderUtils = require('loader-utils')
// a loader
module.exports = function (source) { // 输入
  // 获取开发者配置的本 loader 的 options
  const options = loaderUtils.getOptions(this) // 这个 this 指向谁?
  // some magic code ...
  return content // 输出
  // 或者复杂一点的
  // this.callback(err, content, sourceMap, ast)
  // 必须 return 的是 undefined webpack 才知道你用了 callback
}
```
当然这个 function 也可以写成 async 的
此外 this 上面, loaderUtils 上面都有很多的函数可以使用
更多细节参考[官方文档]()

# Webpack 的 babel-loader 为什么排除 node_modules?
一般情况, 第三方公共 module 应该被 babel 进行过降级处理, 我们没必要把编译好的代码再编译一遍, 降低编译速度

但是有的模块并没有, 这种 module 在被我们项目引用之后 (babel 不管模块加载方式, 只处理 es-next 语法), 如果没有经过 babel 处理 (被 exclude 了), 就可能在某些浏览器环境下 (不支持使用的高级语法) 报错

那么如何处理那些使用了 es-next 语法的模块呢?
  1. 修改 babel-loader 的 exclude, 增加 include, 手动引入需要被 babel 处理的模块
  2. 或者单独用 Babel 编译需要的模块, 然后放进 DLL 里作为一个 entry chunk

与此同时, 想起了一个问题:
之前在给 Vue 项目写 Webpack 配置时遇到问题, 如果不 exclude node_modules 下面的代码, 就会报各种各样的错, exclude 之后就好了

**研究原因**
这是因为, 有一些第三方模块没有提供 esm 的语法 (比如 css-loader, lodash等), 而我们的用 esm 写的业务代码 (或其他一些使用了 esm 语法的第三方模块) 依赖了这些模块, 那么 wepback 在打包时就会将其非 esm 的代码当 esm 模块导入, 这样就可能导致编译时警告或运行时错误

但是啊但是, Webpack 明明可以处理让 ESM 模块引用 CJS 模块, 也可以让 CJS 引用 ESM([参考](../es6/module.u.md)), 为啥 Webpack 会将非 ESM 的代码当做 ESM 模块导入呢? 

**问题出在 Babel**
我们使用 Babel 时会添加 @babel/plugin-transform-runtime 这样的设施来提供 Babel 所需的 helpers 函数, 然而:
  * plugin-transform-runtime 会根据 sourceType 选项来选择注入的 helper 函数是 ESM 或者 CJS, 而 sourceType 的默认值是 module, 那么就会注入成 import
  * 如果一个 module 里面有了 import 这样的代码, Webpack 会判定它是 ESM, 这时候如果它里面又有 module.exports 这样的导出, 或者 require 这样的导入, Webpack 不会为它提供 CJS 的处理方式(即不会管它, 留在打包后的代码里面了就), 然后就会报错

Babel 提供了一个办法来解决这个问题: sourceType：unambiguous
如果 sourceType 选项是 unambiguous 的话, Babel 会根据文件上下文 (比如是否含有 import/export 之类的黑魔法) 来决定用哪种方式注入函数代码:
```js
// babale.config.js
module.exports = {
  ...
  sourceType: 'unambiguous',
  ...
}
```
但是这种做法在工程上不推荐, 因为会对所有编译的文件生效, 同时也增加了编译成本 (unambiguous 会让编译更过程复杂)
而且还有个潜在问题是, 并不是所有 ESM 的模块文件, 都一定含有 import/export 这样的语句, 因而即使某个待编译的文件属于 ESM 模块, 也可能被 Babel 错误的判断为 CJS 模块, 从而引发问题

基于这一点, 一个更合适的做法是, 只对某个我们确定需要进行 Babel 处理的第三方库, 使用 sourceType：unambiguous 选项:
```js
module.exports = {
  ...
  overrides: [ // Babel 的 overrides 选项
    {
      include: './node_modules/module-name/name.common.js', // 使用的具体第三方库
      sourceType: 'unambiguous',
    }
  ]
  ...
}
```

也就是说, 单独处理一个第三方库的话, 需要在 Webpack babel-loader 那把它挑出来, 然后再在 babel 的 config 里给它单独配置一下
结束.
