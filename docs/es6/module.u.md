# 模块化 history

## "假"模块时代
一开始人们用函数表示模块 (我一开始就这样写)
各个函数在同一个文件中, 随意而可能混乱的互相调用, 且存在命名冲突的风险

然后人们用对象表示模块
各个对象里面的属性并不是私有的, 任何开发者都可以随意修改, 这样极易出现 bug

然后人们想起了闭包
闭包简直是一个天生解决数据访问问题的方案, 通过将模块封装为立即执行函数(IIFE)
我们构造了一个私有函数作用域, 再通过闭包, 将需要对外暴露的数据和接口输出
```js
var module = (function ($) {
  var data = 'data'
  function foo() {
    console.log(data)
  }
  function bar(data) {
    data = data
  }
  return { foo, bar }
})(jQuery)
```
但是 IIFE 的模块依然仍然是在一个文件中, 依然不能算作真正的模块化
其实 IIFE 已经具备了 CommonJs 的雏形

## CommonJs
NodeJs 无疑是革命性的, 它首创了 Js 语言体系下的:
  > 一个文件就是一个模块, 有单独的作用域, 对其他文件是不可见的

CommonJs 的特点有:
  文件就是模块, 文件内所有代码运行在独立的(函数)作用域中, 不会污染全局空间
  模块可以被多次加载, 第一次被加载时会被**缓存**, 之后只从缓存中直接读取结果
  模块输出的值通过模块自己的 module.exports 属性被引用
    该属性是输出值的(深)拷贝, 除了取值器函数, 模块内的变化再也不会影响到模块的输出值
    exports 是 module.exports 的引用, 如果直接被指向另一个值不会改变模块的输出值
  模块按代码引入的顺序被加载

一个 CommonJs 简易实现
```js
let module = {} // 模块
module.exports = {} // 模块的输出值

(function (module, exports) {
  // 模块的内容 ...
})(module, module.exports)
```

## AMD 与 CMD 与 UMD
这些都属于 CommonJs 规范, 只是对其在浏览器端实现的一种尝试

因为 NodeJs 运行的时候, 一般所有文件都已经存储在本地了, 所以 CommonJs 的加载过程是同步的
只有模块一个接一个的加载完成, 才会执行后续代码

但是在浏览器环境下, 需要有异步加载的能力, 就出现了 AMD
但是有时候有些模块是需要同步加载的, 于是又出现了 CMD
而整合了各种规范并能根据环境判断使用哪一种的, 就是 UMD

## ES Module
真正面向浏览器的前端模块化方案降临

ESM 相对 CommonJs 的最大区别就是**尽可能的静态化**
这样就可以保证在编译时就确定模块直接的依赖关系, 以及模块输出的变量
而 CommonJs 只能在运行时才能确定这些内容, 这就给基于代码静态分析的 tree shaking 提供了可能

ESM 输出的不是模块输出值的拷贝, 而是它的一个引用:
```js
// data.js
export let data = 'data'
export function modifyData() {
  data = 'modified data'
}

// index.js
import { data, modifyData } from './data'
console.log(data) // => data
modifyData()
console.log(data) // => modified data
```
而在 CommonJs 中的表现如下:
```js
// data.js
var data = 'data'
function modifyData() {
  data = 'modified data'
}
function getData() {
  return data
}
module.exports = {
  data, modifiedData
}

// index.js
const { data, modifyData, getData } = require('./data')
console.log(data) // => data
modifyData()
console.log(data) // => data
console.log(getData()) // => modified data
```

ESM 的静态性也给它带来了一些限制
  只能在文件顶部引入依赖 (除了动态异步 import 函数)
  导出的变量类型受到严格控制
  引入的变量是 ReadOnly 的, 不能重新绑定

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
  加载单一的输出项可以从整体加载的对象中再解构出来 (思考: 这样还可以被 tree-shaking 吗? 不能了)
  
但是一般情况下绝对不要混用两种方式, 因为:
  webpack 在处理时可能出现各种奇怪问题, 主要是 Webpack 配合使用 Babel 引发的问题, [参见](../webpack/webpack.u.md)