https://github.com/mqyqingfeng/Blog/issues/42
https://juejin.cn/post/6844904093467541517

柯里化函数 - 把多参函数变成更少(单或多)参函数的函数

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
  args.length >= fn.length ? fn(...args) : (..._args) => curry(fn, ...args, ..._args)
```