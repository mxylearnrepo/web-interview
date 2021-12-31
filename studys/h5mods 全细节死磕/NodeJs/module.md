https://segmentfault.com/a/1190000023828613

# 头尾包装
我们在一个将要在 Nodejs 环境执行的文件中, 可以直接使用 module, exports, 和 require 以及 __filename, __dirname 这五个变量,
它们来自 Nodejs 对每个 module 的 **头尾包装** 
```js
// 在头部加了
(function (exports, require, module, __filename, __dirname)) {
// ...
// 在尾部加了
});
```

# module 的类型
内置模块: Nodejs 自己的模块, 基于 Libuv/C++
文件模块: 我们写的 Js 模块, 基于内置模块

# require 的顺序
简单来说就是当执行到 require(x) 函数, 根据这个 x
  找内置模块, 内置优先
  找缓存
  将 x 作为路径去找
  将 x 作为路径文件夹去找
  去 node_modules 里面找
  找不到报错
  找到了就执行模板文件, 并缓存 exports

# require 的循环引用
a require 了 b, b 又 require 了 a
当 a.js 加载 b.js, b.js 开始执行, 然后加载 a.js, 
这时 b.js 会得到一个 a.js 的 exports **未完成副本** (或者叫 a 模块的占位 exports)
然后 b.js 执行完成, 产出一个 exports 给 a.js 继续执行最后输出 exports


# 导出的是拷贝 vs 导出的是引用
CommonJs exports 中的对象, 在运行时才会真正生成
而 ESM 模块中的导出是对外的一种静态定义, 是纯语法的东西

CommonJs 在运行时产生了 exports 对象, 这个对象是一个原始值的拷贝
  什么意思呢? 就是说如果赋值给 exports 的是一个引用类型
    那么 require 得到的是原始对象的一个**浅拷贝**
      也就是说, 外部每个模块 require 的是一个原始对象的**不同副本**
而 ESM 模块在静态分析阶段 (不论是在浏览器还是 webpack)
  遇到了 import 就会生成一个 **只读引用**, 这个引用在模块执行后依然指向原来的对象
    就像 Linux 的 symbol link 一样

# 一个容易忽视的问题
看一个例子:
```js
// a.js
let a = 1;
a = a + 1;
module.exports = a;
a = 6;

// index.js
const a = require('./a.js');
console.log(a);  // a的值是2

// b.js
let b = {
  num: 1
};

b.num++;
module.exports = b;
b.num = 6;

// index.js
const b = require('./b.js');
console.log(b);     // { num: 6 }
```
会发现, 在赋值给 exports 的值是基本类型时, 赋值后改变不了它
在赋值给 exports 的值是引用类型时, 它内部属性是可以修改的

# 文章
https://segmentfault.com/a/1190000023828613
https://es6.ruanyifeng.com/#docs/module-loader
http://www.ruanyifeng.com/blog/2015/05/require.html
