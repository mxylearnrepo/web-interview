# 关于 CJS 和 ESM 互相引用这件事
先看阮一峰的 [Node.js 如何处理 ES6 模块](http://www.ruanyifeng.com/blog/2020/08/how-nodejs-use-es6-module.html)

里面说的比较清楚了, 重点有两个:
  > CJS 可以加载 ESM 模块
  ```js
  // 通过 import 表达式 (Nodejs 官网也有介绍)
  (async () => {
    await import('./my.esm.js')
    await import('./my-esm.mjs') // node 的 mjs 格式
  })()
  ```

  > ESM 也可以加载 CJS 模块
  对于普通的 CJS 模块的导出, 需要:
  ```js
  // CJS 导出
  module.exports = Foo

  // ESM 引入
  import Foo from 'foo'
  import * as Foo from 'foo'
  ```
  上面两个 import 导入, 第一个在 Webpack 中会被警告, 而第二种不会
  这是由于两者的语义差异引起的, 第一个需要有 default, 第二个不需要 (暂时的理解是这样)
  而 default 是 ESM 所特有的一个东西

  此外, ESM 导入 CJS 还有一个需要注意的情况:
  ```js
  // 整体加载: 正确
  import packageMain from 'commonjs-package'

  // 单一的输出项: 报错
  import { method } from 'commmjs-package'
  ```
  第二个会报错是因为 ESM 的方案设计上要满足支持静态分析
  而 CJS 的 module.exports 是一个 Js 对象, 无法进行静态分析, 所以只能整体加载
  加载单一的输出项可以从整体加载的对象中再解构出来 (思考: 这样还可以被 tree-shaking 吗?)
  
但是一般情况下绝对不要混用两种方式, 因为:
  webpack 在处理时可能出现各种奇怪问题, 主要是 Webpack 配合使用 Babel 引发的问题, [参见](../webpack/webpack.u.md)