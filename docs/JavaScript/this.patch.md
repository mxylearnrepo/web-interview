# this 指向的规律 (死记硬背)
- (默认绑定) 在函数体中, 非显式或隐式的调用函数时,
	在严格模式下, 函数体内的 this 绑定到 undefined
	在非严格模式下, 函数体内的  this 绑定到 window/global上

- (new 绑定) 使用 new 方法调用构造函数时, 函数体内的 this 绑定到新创建的对象上

- (显式绑定) 使用 call/apply/bind 方式显式调用函数时, 函数体内的 this 会绑定到指定参数的对象上

- (隐式绑定) 通过一个上下文对象调用函数时, 函数体内的 this 绑定到该上下文对象上

- (箭头函数) 箭头函数中, this 指向是由其外层 (函数 or 全局) 作用域来决定的

一句话总结: 除箭头函数是个特殊家伙外, 正常函数按照 new => 显式 => 隐式 => 默认 的顺序逐条规则降级判定

# 实战例题
1. 全局环境中的 this
```js
function f1() {
	console.log(this)
}
function f2() {
	'use strict'
	console.log(this)
}
f1() // window
f2() // undefined
```
这个很简单, 根据第一条铁律可以立即得到答案, 但是如果变种成:
```js
const foo = {
	bar:10,
	fn:function() {
		console.log(this)
		console.log(this.bar)
	}
}
var fn1 = foo.fn
fn1() // window & undefined
```
这里的 this 仍然指向 window , 因为 fn1 的调用, 依然只符合第一条铁律
而 window.bar 没有被定义过, 所以是 undefined
上面这个题目再一次变种:
```js
// foo 还是上面那个 foo
foo.fn() // {bar: 10, fn: f} & 10
```
这个时候, fn 的调用有了一个**执行上下文对象**, 即 foo 对象, 符合第四铁律


2. 上下文对象调用中的 this
