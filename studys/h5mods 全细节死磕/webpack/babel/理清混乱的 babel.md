# babel
> Babel is a JavaScript compiler.

JavaScript 的代码是解释执行的, 那么为什么需要编译? 
  因为很多宿主环境需要降级执行 Js
  因为**自定义 DSL 风格**的出现使各种"姿势"的代码需要编译为 Js

Babel 可以完成:
  一般是 Js 高级特性 (ES Next) 的降级
  源码转换 (比如 JSX, Vue Template 等)
  Polyfill (高级 apis 的实现)

Babel 的设计秉承以下理念:
  可插拔 (第三方开发者力量)
  可调试 (提供 Source Map)
  基于协定 (提供 loose 模式)

然而 Babel 本体 (@babel/core) 并不会直接将一段 es6 代码转换为 es5
它只提供这样一种能力: 将一段任意 ECMA 版本的 js 代码转换为 AST 结构
而要实现工程化所需要的能力, 需要和各种 Babel 插件 及工具 (比如 Webpack) 互相配合

Babel 的 Monorepo 仓库的 packages/ 下面有 140 多个包, 非常庞杂
Babel 家族一些重要的包: 
  - @babel/core
  - @babel/parser
  - @babel/standalone
  - @babel/traverse
  - @babel/types
  - @babel/generator
  - @babel/helper-*
  - @babel/plugin-*
  - @babel/preset-*
  - @babel/preset
  - @babel/cli
  - @babel/polyfill
  - @babel/runtime
  - @babel/runtime-corejs2
  - @babel/runtime-corejs3
  - @babel/node
  - @babel/register
  - ...

@babel/core
Babel 实现转换的核心, 它根据配置, 转换源码:
```js
const babel = require("@babel/core")
babel.transform(code, options, function(err, result) {
  console.log(result) // { code, map, ast }
})
```
而 core 的能力, 又通过 @babel/parser, @babel/code-frame, @babel/generator, @babel/traverse, @babel/types 等使用

一个典型的 Babel 底层编译流程:
```js
const code = "class Example {}"

// 获得 ast
const ast = require("@babel/parser").parse(code, {
  sourceType: 'module',
  plugins: ['jsx', 'flow']
}) // => ast

// 遍历并修改 ast
require("@babel/traverse")(ast, {
  enter(path) {
    // @babel/types 包提供了对具体 ast 节点的修改能力
  }
})

// 聚合 ast
const output = require("@babel/generator")(
  ast, 
  {
    /* options */
    sourceMaps: true
  }, 
  code // 为什么还要传入这个 code?
)
```

@babel/cli
Babel 提供的命令行, 可以在终端中使用 core 的能力

@babel/standalon
面向非 Node.js 环境 (比如浏览器环境) 的编译器

@babel/preset-env
这是一个智能插件集, 有了它基本上我们就能愉快的使用 es6
它会根据我们配置的 targets (一般是浏览器范围或 Node 版本范围), 并结合 core-js 筛选出适配环境所需的 polyfills or plugins, 把它们打包然后提供给我们的业务代码使用

@babel/polyfill 
其实就是 core-js + regenerator-runtime 的结合体
目前已经废弃, regenerator-runtime 好像内置了, 我们的工程需要安装一个 core-js 到项目中

@babel/plugin-transform-runtime
只使用 preset-env 的话, 对于例如 Array.from 等静态方法, 会直接在 global.Array 上增加
对于例如 includes 等实例方法, 会在构造函数的 global.Array.prototype 上添加, 这样就直接修改了全局变量和其原型
如果我们写的库修改了全局变量, 就可能和另一个也修改全局变量的库或使用我们库的业务代码冲突, 所以这样是不太好的
更符合直觉的方式其实应该这样:
```js
var includes = require('xxx/includes')
var array = [1, 2, 3, 4]
includes.call(array, 1)
```
这样的从一个地方去引入, 通过 call 去使用, 就避免了全局污染
plugin-transform-runtime 就解决了这个问题, 其实现当然比上面这个代码复杂一些
这好玩意为啥 preset-env 不内置呢? 非得我们手动添加呢? 咱不知道也不敢问...

此外, Babel core 在编译一段 code 后可能会添加一些内置的 helpers 函数, 比如:
```js
// code class Person()
// 编译后 得到 =====>
function _instanceof(left, right) {
  if (right != null && typeof Symbol !== 'undefind' && right[Symbol.hasInstance]) {
    return !!right[Symbol.hasInstance](left)
  } else {
    return left instanceof right
  }
}

function _classCallCheck(instance, Constructor) {
  if (!_instanceof(instance, Constructor)) {
    throw new TypeError("Cannot call a class as a function")
  }
}

var Person = function Person() {
  _classCallCheck(this, Person)
}
```
这样的帮助函数如果在每一个 class 被编译后都加入, 就会变得非常多
在启用 @babel/plugin-transform-runtime 后, 上述编译结果会变为:
```js
var _interopRequireDefault = require("@babel/runtime/helpers/interopRequireDefault")
var _classCallCheck2 = _interopRequireDefault(require("@babel/runtime/helpers/classCallCheck"))
var Person = function Person() {
  (0, _classCallCheck2.default)(this, Person);
};
```
这里又牵扯出了一个 @babel/runtime 包

@babel/runtime
这里面包含了 Babel 编译时所需要的一切运行时 helpers

对于这俩个包:
  @babel/plugin-transform-runtime 需要和 @babel/runtime 配合着使用
  @babel/plugin-transform-runtime 引用 @babel/runtime 里的 helpers 来缩减代码
  @babel/plugin-transform-runtime 是一个开发时依赖, @babel/runtime 是一个运行时依赖
  @babel/plugin-transform-runtime 与 @babel/runtime 结合使用还可以避免一些编译产生的全局对象污染全局作用域
至于怎么做的, 就得看源码去了解了

@babel/plugin
Babel 插件集合:
  @babel/plugin-syntax-* 是 Babel 的语法插件, 可以扩展 @babel/parser 的一些能力
  @babel/plugin-proposal-* 是用于编译转换再议阶段的语言特性
  @babel/plugin-transform 是 Babel 的转换插件 (@babel/plugin-transform-runtime)

此外还有很多包, 就不继续列举了
总之, Babel 的包分工协调和组织非常庞杂混乱, 要做到真正理解并非一夕之功


# Babel 工程化启示
Babel 生态和前端工程中的各个环节都是打通开放的
它可以 babel-loader 的形式和 Webpack 协作, 也可以 @babel/eslint-parser 的形式和 ESLint 合作

前端工程是一环扣一环的, 作为工具链上任意一环, 插件化能力是设计的重点和关键

# 统一标准化的 babel-preset
待研究...
