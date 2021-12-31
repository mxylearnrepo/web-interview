# this 的指向
(死记硬背)
- (默认绑定) 在函数体中, 非显式或隐式的调用函数时,
	在严格模式下, 函数体内的 this 绑定到 undefined
	在非严格模式下, 函数体内的  this 绑定到 window/global上

- (new 绑定) 使用 new 方法调用构造函数时, 函数体内的 this 绑定到新创建的对象上

- (显式绑定) 使用 call/apply/bind 方式显式调用函数时, 函数体内的 this 会绑定到指定参数的对象上

- (隐式绑定) 通过一个上下文对象调用函数时, 函数体内的 this 绑定到该上下文对象上

- (箭头函数) 箭头函数中, this 指向是由其外层 (函数 or 全局) 作用域来决定的

一句话总结: 除箭头函数是个特殊家伙外 (可以看做最高优先级), 正常函数按照 new => 显式 => 隐式 => 默认 的顺序逐条规则降级判定

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
要点:
  除了 this, 还可以传入更多参数, 作为返回新函数的参数
  创建的新函数可能又传入多个参数
  新函数可以被用作构造函数使用
  新函数可能有返回值
```js
Function.prototype.bind = Function.prototype.bind || function (context, ...args) {
  const self = this

  const bound = function (..._args) {
    context = this instanceof F ? this : context || this
    return self.apply(context, args.concat(_args))
  }

  // 让 bound 接在 this 函数的原型链后面, 因为 new 出来的对象原型链上得有 this
  bound.prototype = Object.create(this.prototype)
  return bound
}
```

# apply 实现
要点:
  context 可能传入 null
  args 是一个数组 (可为空)
  函数本身可能有返回值
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
call 传入的是不固定个数个参数, 只需要将 apply 中的参数 args 数组换成 ...args 即可

# new 实现
要点:
  产生的新对象要进行[原型链继承](extends.md)
  构造函数如果显式返回一个对象数据类型则返回它, 不然返回新创建的对象
  而如果返回的是一个基本数据类型, 返回的就还是 new 新创建的那个 obj
```js
// 里面不能用 new 哦 ~
function newFunc(constructor, ...args) {
  const obj = Object.create(constructor.prototype)
  const result = constructor.apply(obj, args) // 绑定 this
  return (result && typeof result === 'object') ? result || obj : obj
}
```
可见这个玩意类似于一个组合继承, 只有细微差别
需要注意的的是, 如果构造函数中返回的是一个对象 (复杂类型), 那么 instance 就是这个对象

# Object.create 实现
```js
function objectCreate(prototypeObj) {
  if (typeof proto !== 'object' && typeof proto !== 'function') {
    throw new TypeError('Object prototype may only be an Object or null.')
  if (propertyObject == null) {
    new TypeError('Cannot convert undefined or null to object')
  }
  const F = new Function() // function F() {}
  F.prototype = prototypeObj
  const obj = new F() // obj.__proto__ === prototypeObj

  if (!prototypeObj) {
    obj.__proto__ = null
  }

  return obj
}
```
所以 Object.create 就是给 obj.__proto__ 赋值, 为什么不能直接这样?
```js
function objectCreate(prototypeObj) {
  const obj = {} // Object()
  obj.__proto__ = prototypeObj && (typeof prototypeObj === 'object' || typeof prototypeObj === 'function') ? prototypeObj : null
  return obj
}
```
