# Code Splitting
以前有个 CommonsChunk 和 CommonsChunkPlugin, 现已在 webpack 4 版本后被废弃
取而代之的是 SplitChunks

首先现在的 webpack 提供了 3 种实现 code splitting 的方法:
  配置多个 entry: 多个 entry 表示多个入口文件, 也就有多个构建目标
  抽取公共的代码: 使用 splitChunks 配置抽取公有代码
  按需动态懒加载: 动态 import 加载一些代码

## splitChunks
这个东西是由 webpack 内置的 SplitChunksPlugin 提供的能力, 可直接在 optimization 中配置
```js
// optimization.splitChunks 的默认配置
module.exports = {
  //...
  optimization: {
    splitChunks: {
      chunks: 'async', // 表示从哪些 chunks 里抽取代码, 可以是 initial, async, all 或一个函数
      minSize: 30000, // 抽取出来的文件在压缩前的最小大小, 默认为 30000, 太小不抽取
      maxSize: 0, // 抽取出来的文件在压缩前的最大大小, 默认为0, 表示不限制最大大小
      minChunks: 1, // 表示被不同 chunks 引用次数, 默认为 1, 表示被引用过一次的代码就可以抽取
      maxAsyncRequests: 5, // 最大的按需(动态)加载次数, 默认为 5, 超过就不新增
      maxInitialRequests: 3, // 最大的初始化加载次数 默认为 3, 
      automaticNameDelimiter: '~', // 抽取出来的文件的自动生成名字的分隔符, 默认为 ~
      name: true, // 抽取出来的文件的名字, 默认为 true, 表示自动生成文件名
      cacheGroups: { // 缓存组(奇怪的名字, 应该叫抽取组吧), 上面的可以都不用管, 这才是我们配置的关键
        vendors: { // 会将多个 chunks 中所有引入 node_modules 的文件打包成一个大的 单独的 chunk
          test: /[\\/]node_modules[\\/]/,
          priority: -10
        },
        default: { // 除 node_module 之外, 如果多个 chunks 引入相同模块 >=2 就会拆包
          minChunks: 2,
          priority: -20,
          reuseExistingChunk: true
        }
      }
    }
  }
}
```
cacheGroups 可以继承/覆盖上层 splitChunks 中所有的参数值, 除外还有几个特有配置项
  test: 表示要过滤 modules, 类似于 loaders 的 test, 默认为所有 modules, 可以是正则或函数
  priority: 表示抽取模块的权重, 如果一个 module 同时满足多个 cacheGroup 项, 则权重高的说了算
  reuseExistingChunk: 为 true 则表示如果当前的 chunk 包含的模块有已经抽取出去的, 将不会重新生成新的
  ...其他的先不去管

## 一个思考
所以 splitChunks 这个东西的前提是我们已经有多个 chunks 了, 需要从这么多 chunks 中抽取公共 chunks 对吧
那么,
有些常规单页应用, 一般只会有一个入口文件打包出一个 chunk (bundle.js), 里面不同模块 a 和 b 引用了共同的模块 c
这时候 a, b, c 都会打包到一个 bundle 里面, c 并不会拆分成单独的 chunk 文件, splitChunks 配置对于这种是不起作用的
这时候需要拆包就只有通过动态加载的方式, 形成 bundle (initial) + async 的不同 chunks, 才可以了就

## 一些例子
> 把所有 node_modules 的模块被不同的 chunk 引入超过 1 次的抽取为 common:
```js
cacheGroups: {
  common: {
    test: /[\\/]node_modules[\\/]/,
    name: 'common',
    chunks: 'initial',
    priority: 2,
    minChunks: 2,
  },
}
```
> 把一些基础的框架单独抽取如 react:
```js
cacheGroups: {
  reactBase: {
    name: 'reactBase',
    test: (module) => {
      return /react|redux|prop-types/.test(module.context)
    },
    chunks: 'initial',
    priority: 10,
  },
  common: { // 其他的抽取成 common
    name: 'common',
    chunks: 'initial',
    priority: 2,
    minChunks: 2,
  },
}
```
但是有点问题, 简单的用正则匹配 react 可能造成误杀, 比如有些 react 组件命名就带 react 这个词

> css 代码也可以通过 splitChunks 抽取公共:
```js
cacheGroups: {
  styles: {
    name: 'styles',
    test: /\.css$/,
    chunks: 'all',
    enforce: true, // 这是个啥?
    priority: 20, // 注意优先级必须最高, 不然可能其他 cacheGroups 项会提前打包一部分样式文件
  }
}
```

# 更多更详细
https://webpack.docschina.org/guides/code-splitting/
https://webpack.docschina.org/plugins/split-chunks-plugin/