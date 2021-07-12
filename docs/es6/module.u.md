# 关于 CJS 和 ESM 互相引用这件事
先看阮一峰的 [Node.js 如何处理 ES6 模块](http://www.ruanyifeng.com/blog/2020/08/how-nodejs-use-es6-module.html)

里面说的比较清楚了, 重点有两个:
  - CJS 可以加载 ESM 模块
  ```js
  // 通过 import 表达式 (Nodejs 官网也有介绍)
  (async () => {
    await import('./my.esm.js')
    await import('./my-esm.mjs') // node 的 mjs 格式
  })()
  ```
  - ESM 也可以加载 CJS 模块
  ```js
  // 整体加载: 正确
  import packageMain from 'commonjs-package'

  // 单一的输出项: 报错
  import { method } from 'commmjs-package'
  ```
  第二个会报错是因为 ESM 的方案设计上要满足支持静态分析
  而 CJS 的 module.exports 是一个 Js 对象, 无法进行静态分析, 所以只能整体加载
  加载单一的输出项可以从整体加载的对象中再解构出来 (思考: 这样还可以被 tree-shaking 吗?)
  