# this 的指向
[穿越](../../docs/JavaScript/this.u.md)
其中, 显式绑定 (bind apply call) 其实就是封装的隐式绑定
bind 绑定的函数作为构造函数被 new 时, 会被 new 修改 this 指向
而箭头函数中的 this 无法被显式绑定更改
而且箭头函数不是构造函数, 自然也不能进行 new 绑定
```js
function foo() {
  return () => {
    console.log(this.a)
  }
}

const obj1 = { a: 1 }
const obj2 = { a: 2 }

console.log(foo.call(obj1).call(obj2)) // 1
```

# bind 实现
```js
Function.prototype.bind = Function.prototype.bind || function (context, ...args) {
  const self = this
  const F = new Function()
  const bound = function (..._args) {
    context = this instanceof F ? this : context || this
    return self.apply(context, args.concat(_args))
  }

  F.prototype = this.prototype // 调用 bind 的函数的 prototype 使原型链完整
  bound.prototype = new F() // 让 bound 作为构造函数时 原型链上有 F 函数
  // 不进行 bound.prototype.constructor = bound 修复
  // 原因是这样就可以使 this.prototype.constructor 作为 bound 的实例的构造函数
  return bound
}
```

# apply 实现
```js
Function.prototype.apply = Function.prototype.apply || function (context, args) {
  args = args || []
  context = context || window

  const fnKey = Symbol('fnKey')
  context[fnkey] = this
  
  const res = context[fnKey](...args)
  delete context[fnKey]
  return res
}
```

# call 实现
只需要将 apply 中的参数 args 数组换成 ...args 即可

# new 实现
```js
// 里面不能用 new 哦 ~
function newFunc(constructor, ...args) {
  const obj = Object.create(constructor.prototype)
  const result = constructor.apply(obj, args) // 绑定 this
  return (result && typeof result === 'object') ? result : obj
}
```
配合[apply](#apply-实现)一起看
