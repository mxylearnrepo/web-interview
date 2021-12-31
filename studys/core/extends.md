关于[原型](../../docs/JavaScript/prototype.u.md)
[如何优雅的实现继承](../../studys/code/inherit.md)
[乱七八糟继承](https://github.com/mqyqingfeng/Blog/issues/16)
[JavaScript：继承和原型链（译）](https://justjavac.com/2015/12/09/inheritance-and-the-prototype-chain.html)

# 实现一个 instanceof
instanceof 就是判断某一个构造函数的 prototype 属性是否出现在了一个实例的原型链上
```js
function instanceOf(obj, constructor) {
  let proto = obj.__proto__ // 写法可换成 Object.getPrototypeOf(obj)
  while(true) {
    if (proto === null) return false // 最后必然到 null
    if (proto === constructor.prototype) return true 
    proto = proto.__proto__
  }
}
```
生动的说明了 所谓原型链就是一个对象 (任何对象都是) 的 ...__proto__.__proto__.__proto__... 链
而访问**一个对象的属性**, 就是在这个链上一层一层往上找, 找到或 __proto__ 为 null 时为止

# 继承
继承的说法不准确, 委托应该更准确些

阶段一 (直接修改原型)
```js
function inherit(Child, Parent) {
  Child.prototype = new Parent()
  Child.prototype.constructor = Child // 修正原型链上的构造函数
}
// 简单直接, 但是呢, 不同的 Child 实例的 __proto__ 会指向同一个 Parent 的实例
// 同时 Child 在创建 (new) 实例时不能向 Parent 这个构造函数传参
```

阶段二 (借助构造函数)
```js
function Child(...args) {
  Parent.call(this, args)
}
// 这个就有意思了, 每初始化一个 Child 实例, 就把 Parent 构造函数以 Child 实例身份执行一次
// 但是问题很大, Parent 原型的方法在 Child 实例上并不存在 (Parent.prototype.xxx)
// 但是 Child 可以向 Parent 传参了
```

阶段三 (组合继承)
```js
function Child(arg1, arg2) {
  this.arg2 = arg2
  Parent.call(this, arg1)
}
function inherit(Child, Parent) {
  Child.prototype = new Parent()
  Child.prototype.constructor = Child

  // 带静态属性的组合继承
  Child.__proto__ = Parent
}
// 好像就是把上面的组合起来了, 为啥要这样搞呢?
// 其实就把上面一和二的优点集成了一下, 当然它们问题依然还是存在的
// 但不妨碍组合继承成为 Js 中最常用的继承模式, 因为它简单, 并且有效
// 主要问题就在于调用了两次 Parent 构造函数
```

阶段四 (寄生式组合继承)
可以解决调用两次 Parent 构造函数的问题
```js
function inherit(Child, Parent) {
  Child.prototype = Object.create(Parent.prototype)
  Child.prototype.constructor = Child
}
// 所以就是不直接 Child.prototype = new Parent()
// 而是通过一个中间对象来产生 Parent 的原型的副本
```

阶段五 (使用 es6 class)
```js
class Parent { 
  constructor(args) {}
}
class Child extends Parent {
  constructor(args) { super(args) }
}
// 到这一步, 依然是不完全符合面向对象思想的, 但已经相对比较优雅了
```

# 剖析 Babel 的实现
在 ES6 时代我们怎么使用继承
```js
class Parent {
  constructor() {}
}
class Child extends Parent {
  constructor() { super() } // super 是一个关键字, 子类中必须调用, 内部 this 是 Child 的实例
}
```
[关于关键字 super](https://www.jianshu.com/p/fc79756b1dc0)

上面的代码大致会被 Babel 编译成这个样子
```js
// 父类
var Parent = function Parent() {
  _classCallCheck(this, Person) // 使用检测
}
// 子类
var Child = (function (_Parent) {
  _inherits(Child, _Parent) // 一次原型链继承
  function Child() { // 定义 Child
    _classCallCheck(this, Child) // 使用检测
    // _get(Object.getPropertyOf(Child.prototype), 'constructor', this).call(this)
    // _get 内部逻辑十分他妈的复杂, 这行代码做的事情其实类似于
    _Parent.call(this)
  }
  return Child // 返回这个 Child 构造函数
})(Person) // 依赖 Person

function _inherits(subClass, superClass) {
  // 创建继承关系
  subClass.prototype = Object.create(superClass && superClass.prototype, {
    constructor: { // 注意: 也可以这样修正 constructor
      value: subClass, enumerable: false, writeable: true, configurable: true
    }
  })

  // 添加静态方法
  superClass && Object.setPropertyOf ? Object.setPropertyOf(subClass, superClass) : subClass.__proto__ = superClass
}
```
可见其原理与上面的组合继承方案基本一致
