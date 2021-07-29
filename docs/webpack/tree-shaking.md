# Tree Shaking 为什么要依赖 ESM 规范？
tree-shaking 的大前提是 esm 的静态性:
  - import 导入的模块名只能是字符串常量
  - import (一般) 必须出现在 js 文件顶部
  - export default 的导出不可以解构 (对 tree-shaking 似乎没什么帮助呢)
  - ...

而 CommonJS 定义的模块化规范, 只有在代码执行后, 才能自行确定依赖模块

## Tree Shaking 友好
看一段代码
```js
// 模块一
export default {
	add(a, b) {
		return a + b
	}
	subtract(a, b) {
		return a - b
	}
}
// 模块二
export class Number {
	constructor(num) {
		this.num = num
	}
	add(otherNum) {
		return this.num + otherNum
	}
	subtract(otherNum) {
		return this.num - otherNum
	}
}
```
这俩种模块的导出方式下, Webpack 都会保留整个导出对象, 不能把类 or 对象的内部没用到的方法和属性消除掉
原因是类和对象这种东西都是动态的, 可能在执行时里面的东西才生成, 且里面的东西也都可以变, 没有静态单独的 export 方便去保证正确匹配 

所以问题的关键似乎在**不用 export 单独导出**就不容易找到导出对象里**究竟有些什么**, 但应该还是有办法的吧?
现在有一些工具能做对于对象 or class 的方法属性的裁剪 (比如 webpack-deep-scope-plugin 可能基于复杂的程序流分析算法)
但是都不一定完全靠谱, 而且增加了编译时负担, 所以不太有必要使用

所以可以总结一个:
  不要导出 一个 包含多项属性和方法的 对象 or class
  不要使用 export default 导出模块内容

所以 Tree Shaking 优化友好的一个最佳实践就是, 原子化和颗粒化的 **export 单独导出纯函数**, 像这样:
```js
export function add(a, b) {
  return a + b
}
export function substract(a, b) {
  return a - b
}
```
但话说回来, 很多时候我们单独导出的函数不一定是纯函数, 那么还是会因为"副作用"导致某些代码不能被"摇掉"
这时候需要设置 sideEffects 来实现完整的 tree-shaking

## sideEffects
tree shaking 不能"摇掉"模块中的[副作用代码](../JavaScript/functional_programming.u.md)
为啥呢? 因为**工具只知道这段代码是"可能有副作用的", 并不能判断出是不是真的有副作用 (或者是该副作用是不是真的有什么影响)**
为了解决这种"有副作用的"模块无法进行 tree shaking 优化的问题, Webpack 给出的方案是, 我们在**被引入模块所在目录**的 package.json 中加入一个 sideEffects 字段来告诉 Webpack 哪些模块可以被认为无副作用:
```js
// 代表所有模块均无副作用
{
  "name": "your-project",
  "sideEffects": false
}
// 或者
// 代表哪些模块有副作用 (其他就认为没有)
{
  "name": "your-project",
  "sideEffects": [
    "./src/some-side-effectful-file.js"，
    "*.css"
  ]
}
```
**关于 Babel**
有的人说因为 babel 会在构建工具处理前将我们的 esm 代码默认地转换为 cjs, 导致构建工具不能 tree-shaking
以前确实是这样的, 但是现在的 babel-loader 已经关闭了默认转换 esm 的开关 (但是 babel 本身还是默认转换成 cjs)

那为什么很多时候 tree-shaking 还是不好使呢? 这是因为 babel 在降级处理我们的代码时还是会引入新的"副作用"
而且我们自己写的代码也很难保证没有"副作用" (即使我们知道没有, 也不能保证构建工具知道没有)
所以最终还是加上一个 sideEffects: false 属性, 应该就好使了

## Webpack 怎么处理 Tree Shaking
Webpack 会在 production mode 下自动开启 tree-shaking, 实际上真正依赖的是 TerserPlugin, UgilifyJs 提供的能力
Webpack 在分析 (ast parse) 我们的代码时, 产生三类标记:
  harmony export, 标记*被使用了*的 export
  unused harmony export, 标记*没有被使用*的 export
  harmony import, 标记*所有*的 import
然后这些压缩插件就会根据这些信息去删除代码了
所以无用代码能不能被"摇掉"的本质原因是 Webpack 能标记出它们
在 development 模式下也可以开启 tree shaking:
```js
const config = {
  mode: 'development',
  optimization: {
    usedExports: true,
    minimizer: [
      new TerserPlugin({...}) // 支持删除死代码的压缩器
    ]
  }
}
```

## Vue 和 Tree Shaking
假如我们 import 了 Vue 进来, 但是没有用它里面的一些静态方法比如 Vue.nextTick, Vue.set 等, 那根据上文这些代码是不能"摇掉"的
如果我们通过具名引用 import 具体的方法, 或者用 import * as, 就可以被"摇掉"了, 当然这个前提是 Vue 这样的公共库支持这样引入
```js
// 不能 tree shaking
import Vue from 'vue'
Vue.nextTick(() => {})
// 可以 tree shaking
import { nextTick } from 'vue'
nextTick(() => {})
// 可以 tree shaking
import * as Vue from 'vue'
nextTck(() => {})
```

## Lodash 和 Tree Shaking
lodash 这个库非常有意思, 它本体打包结果不支持 tree shaking, 打包出来的代码是 UMD 的
于是它又支持了打包出一个 ESM 规范的 lodash-es:
```js
// lodash 构建成 lodash-es 的 package.json
{
  "main": "lodash.js",
  "module": "lodash.js",
  "name": "lodash-es",
  "sideEffects": false,
  //...
}
```
啥字段都有, 给了 Webpack 充足的信息

## CSS 和 Tree Shaking
CSS 也能摇, 要找出没用到的选择器样式, 删除掉, 其实原理和找到没用的 Js 代码是一样的
所不同的是, Webpack 遍历 Js 代码用的是 acorn, 遍历 CSS 就需要 PostCSS 了
当然, Webpack 本身不会去做这件事, 它只负责遍历 Js, 面对 *.css 需要上 loader 和 plugin
实现这一功能的插件是 purgecss-webpack-plugin, 其执行过程是:
  - 监听 Webpack 编译完成, 从 compilation 对象中找到所有的 CSS 文件代码 
  - 将所有的 CSS 文件经由 PostCss 过一遍拿到 AST 数据结构
  - 遍历 AST 找出无用的 CSS 选择器样式代码, 删除之
  - 重新组合成 CSS 文件字符串交还给 Webpack
