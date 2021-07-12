需要注意的是, 如果 构造函数中出现了显式 return 的情况
```js
function Foo() {
	this.user = 'Maos'
	const o = {}
	return o
}
const instance = new Foo()
console.log(instance.name) // undefined
```
这是因为如果构造函数中返回的是一个对象 (复杂类型), 那么 instance 就是这个对象
o = {} 里面没有 user 属性所以输出 undefined
```js
function Foo() {
	this.user = 'Maos'
	return 1
}
const instance = new Foo()
console.log(instance.name)
```
而如果返回的是一个普通值 (基本类型), instance 就还是 new 新创建的那一个 obj
这一点也可以从 new 的实现中推断出来