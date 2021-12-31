# Scope Hoisting
作用域提升 是指 webpack 通过 ES Module 静态分析依赖, 尽可能的把存在单独的依赖关系的模块放到同一个函数里
我们知道 webpack 打包没有程序流分析, 所以构建结果像一个模块映射表, 每个模块封装成单独的一个函数
我感觉这个 scope hoisting 就是 webpack 尽量向 rollup 更小的构建结果看齐的一种尝试, 尽力的把一些模块合并到一起

配置也在 optimization 里:
```js
// webpack.config.js
module.exports = {
  optimization: {
    concatenateModules: true
  }
}
```