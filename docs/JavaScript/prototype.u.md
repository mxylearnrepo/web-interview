# 原型
## 什么是"一个对象的原型"
每个对象内部都有一个"内部链接"指向另一个对象, 这个就是它的原型
原型对象也是对象, 也有自己的原型, 如此直到某个对象以 null 作为其原型, 这就是原型链
null 作为原型链的末尾存在 (元始天尊)

## 怎么访问"一个对象的原型"
我们随便给一个对象 let a = {} 访问 a.prototype 得到的是 undefined
为什么会这样呢? 因为 a.prototype 本来就没定义过

不是每个对象都有原型吗? 那么 a 的原型对象在哪?
并不在它自己的 prototype 属性上 (虽然 chrome 有补全但是普通对象真的没有这个属性)
只有函数对象 (type === 'function') 才有 prototype 属性
```js
function doSomething(){}
console.log(doSomething.prototype)

// 控制台输出的原型对象:
{
  constructor: ƒ doSomething(),
  __proto__: Object
}
```
可以看到原型对象的结构, 一个 constructor 和一个 __proto__
constructor 就是这个函数自己, 这个很好理解
__proto__ 则指向任何一个对象 (函数对象或普通对象) 的构造函数的原型对象 (prototype)

任何对象都是来自 Object 构造函数, 而 Object.prototype.__proto__ === null (末尾)
任何函数对象都是来自 Function 构造函数, 而 Function 是由 Object 创造的
Object 就像一个创世之神, 给它创造的每一个对象里放一个指向自己的 prototype 的 __proto__ 属性
但是 Object 是允许创造出的对象改变 __proto__ 属性值的, 因为不管怎么变, 都得有一个对象承担这个值, 最终都会回到自己

Function 是唯一的一个特殊的构造函数
```js
Function.__proto__ === Function.prototype // 内置匿名函数 `anonymous`
Function.__proto__ === doSomething.__proto__
```
等于是 Object 把 Function 创造出来, 又允许它不认自己为爸爸, 而是认一个内置匿名函数 anonymous 为爸爸
然后 Function 又学着 Object 给了自己一个 prototype, 但是连同 Function.__proto__ 一起指向了自己的伪爸爸

然后 Function 又说 Object 也是一个构造函数, 但是它是自己亲爸爸, 创世神, Function 管不了
但是所有其他由 Function 自己创造的函数对象, 它可管得了, 于是让所有函数对象里都必须有一个 prototype 对象

这个 prototype 对象 Function 无力创造 (它只能创造函数对象), 所以还得由亲爸爸 Object 代为创建, 
所以 prototype 对象里面就有这个函数对象它自己 (constructor), 又有一个指向亲爸爸的 prototype 的 __proto__

同时这些由 Function 自己创造的函数还必须有一个 __proto__ 指向自己的伪爸爸 (等于 Function.prototype)
感觉 Function 就是一个带孝子啊
而 Object 像一个宽容的造物者

而所谓的 原型链 就是对象的 __proto__.__proto__.__proto__... 组成的链
任何一个对象 (除了 null) 都有一个 __proto__ 属性

## 以原型链实现的继承
继承的说法不准确, 委托应该更准确些

## 创建对象的不同方式
[JavaScript：继承和原型链（译）](https://justjavac.com/2015/12/09/inheritance-and-the-prototype-chain.html)
