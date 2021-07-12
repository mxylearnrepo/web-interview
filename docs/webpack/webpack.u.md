# Webpack 的 babel-loader 为什么排除 node_modules?
一般情况, 第三方公共 module 应该被 babel 进行过降级处理, 我们没必要把编译好的代码再编译一遍, 降低编译速度

但是有的模块并没有, 这种 module 在被我们项目引用之后 (babel 不管模块加载方式, 只处理 es-next 语法), 如果没有经过 babel 处理 (被 exclude 了), 就可能在某些浏览器环境下 (不支持使用的高级语法) 报错

那么如何处理那些使用了 es-next 语法的模块呢?
  1. 修改 babel-loader 的 exclude, 增加 include, 手动引入需要被 babel 处理的模块
  2. 或者单独用 Babel 编译需要的模块, 然后放进 DLL 里作为一个 entry chunk

与此同时, 想起了一个问题:
之前在给 Vue 项目写 Webpack 配置时遇到问题, 如果不 exclude node_modules 下面的代码, 就会报各种各样的错, exclude 之后就好了

**研究原因**:
这是因为, 有一些第三方模块没有提供 esm 的语法 (比如 css-loader, lodash等), 而我们的用 esm 写的业务代码 (或其他一些使用了 esm 语法的第三方模块) 依赖了这些模块, 那么 wepback 在打包时就会将其非 esm 的代码当 esm 模块导入, 这样就可能导致编译时警告或运行时错误

但是啊但是, Webpack 明明可以处理让 ESM 模块引用 CJS 模块, 也可以让 CJS 引用 ESM, 为啥 Webpack 会将非 ESM 的代码当做 ESM 模块导入呢? ...问题出在 Babel

我们使用 Babel 时会添加 @babel/plugin-transform-runtime 这样的设施来提供 Babel 所需的 helpers 函数, 然而:
  * plugin-transform-runtime 会根据 sourceType 选项来选择注入的 helper 函数是 ESM 或者 CJS, 而 sourceType 的默认值是 module, 那么就会注入成 import
  * 如果一个 module 里面有了 import 这样的代码, Webpack 会判定它是 ESM, 这时候如果它里面又有 module.exports 这样的导出, 或者 require 这样的导入, Webpack 不会为它提供 CJS 的处理方式, 然后就会报错

Babel 提供了一个办法来解决这个问题: sourceType：unambiguous
如果 sourceType 选项是 unambiguous 的话, Bable 会根据文件上下文 (比如是否含有 import/export 之类的黑魔法) 来决定用哪种方式注入函数代码:
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

基于这一点, 一个更合适的做法是, 只对某个确定需要进行 Babel 处理的第三方库, 使用 sourceType：unambiguous 选项:
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
