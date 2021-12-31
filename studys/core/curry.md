https://github.com/mqyqingfeng/Blog/issues/42
https://juejin.cn/post/6844904093467541517

# 柯里化函数
把多参函数变成更少(单或多)参函数的函数

比如一个 add 函数它可以同时支持:
```js
const add = curry((a, b, c) => (a + b + c))
add(1, 2, 3)
add(1, 2)(3)
add(1)(2, 3)
add(1)(2)(3)
```
实现 curry:
```js
const curry = (fn, ...args) => 
  args.length >= fn.length ? 
  fn(...args) : 
  (..._args) => curry(fn, ...args, ..._args)
```

# 偏函数
和柯里化很像, 将一个 n 参的函数转换成固定 x 参的函数, 剩余参数将在下次调用全部传入
```js
const partial => (fn, ...args) => (..._args) => fn(...args, ..._args)
```
