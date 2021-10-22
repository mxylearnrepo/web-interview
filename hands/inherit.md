[乱七八糟继承](https://github.com/mqyqingfeng/Blog/issues/16)

# 继承
阶段一 (直接修改原型)
```js
function inherit(Child, Parent) {
  Child.prototype = new Parent()
  Child.prototype.constructor = Child // 修正原型链上的构造函数
}
// 简单直接, 但是呢, 不同的 Child 实例的 __proto__ 会指向同一个 Parent 实例
// 同时 Child 在创建实例时不能向 Parent 传参
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
}
// 好像就是把上面的组合起来了, 为啥要这样搞呢?
// 其实就把上面一和二的优点集成了一下, 当然它们问题依然还是存在的
// 但不妨碍组合继承成为 Js 中最常用的继承模式, 因为它简单, 并且有效
```

阶段四 (带静态属性的组合继承)
```js
function inherit(Child, Parent) {
  Child.prototype = new Parent()
  Child.prototype.constructor = Child

  // 存储超类
  Child.super = Parent
  Child.__proto__ = Parent
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
