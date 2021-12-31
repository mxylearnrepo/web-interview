# core-js
core-js 是一个 JavaScript 标准库, 涵盖了 ECMAScript 2020 在内的众多特性的 polyfills
作者是一位战斗民族彪悍大叔, 曾因飚摩托车超速撞人致死被判入狱 18 个月, 并处罚金 138 万卢布

---

core-js 使用
我们可以这样用:
```js
import 'core-js' // polyfills 就进来了
```
或者这样用:
```js
import 'core-js/features/array/from' // 只引入某一个 polyfill
```

core-js 项目一览
core-js 也是一个由 lerna 搭建的 monorepo 风格的工程, 它的 packages 下面有五个包:
 - core-js          逻辑核心, 提供了基础的垫片能力
 - core-js-pure     提供不污染全局变量的垫片能力
 - core-js-compact  按照 browserslist 需要提供垫片能力
   ```js
   // 筛选出全球使用份额 > 2.5% 的浏览器范围, 并提供在这个范围下需要使用的垫片能力
   const {
     { 
       list,   // array of required modules
       targets // object with targets for each module
     } = require('core-js-compact')({ targets: '> 2.5%' })
   }
   ```
 - core-js-builder  打包出 core-js 代码
   ```js
   // 把符合需求的垫片打包到 my-core-js=built.js 中
   require('core-js-builder')({
     targets: '> 0.5%',
     filename: './my-core-js-bundle.js',
   }).then(code => {}).catch(err => {})
   ```
 - core-js-bundle  打包好的 core-js 代码?
   ```js
   require('./packages/core-js-builder')({
     filename: './packages/core-js-bundle/index.js',
   }).then(done).catch(err => {
     console.err(err)
     process.exit(1)
   })
   ```
整齐划一, 非常 nice, 宏观上的设计, 体现了工程复用能力
